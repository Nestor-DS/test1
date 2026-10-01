```

package mx.com.inbursa.quimera.core.service;

import java.io.IOException;
import java.net.URI;
import java.util.Collections;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Locale;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * Control local de abuso por IP.
 *
 * Compatible con Java 7 y javax.servlet.
 * El estado se comparte entre peticiones de esta instancia/JVM,
 * no entre Managed Servers.
 *
 * No sustituye autenticacion, autorizacion ni proteccion CSRF.
 * No confiar en X-Forwarded-For sin configurar un proxy de confianza.
 */
public class AbuseGateFilter implements Filter {

    static final int MAX_REQUESTS = 60;
    static final long WINDOW_MS = 60_000L;
    static final long BAN_MS = 5 * 60_000L;
    static final int MAX_QUERY_LENGTH = 2048;
    static final int MAX_KEYS = 100_000;
    static final long CLEANUP_EVERY_MS = 60_000L;

    private static final int MAX_URI_LENGTH = 4096;

    private static final Set<String> ALLOWED_ORIGINS;

    static {
        Set<String> origins = new HashSet<String>();
        origins.add("https://app.inbursa.com");
        origins.add("https://quimera.inbursa.com");
        ALLOWED_ORIGINS = Collections.unmodifiableSet(origins);
    }

    private final ConcurrentHashMap<String, Window> windows =
            new ConcurrentHashMap<String, Window>();

    private final AtomicLong lastCleanup =
            new AtomicLong(System.currentTimeMillis());

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // Sin inicializacion adicional.
    }

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        if (!(request instanceof HttpServletRequest)
                || !(response instanceof HttpServletResponse)) {
            chain.doFilter(request, response);
            return;
        }

        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse res = (HttpServletResponse) response;

        if (looksGarbage(req)) {
            reject(res, HttpServletResponse.SC_BAD_REQUEST,
                    0L, "bad request");
            return;
        }

        /*
         * Los recursos estaticos no consumen el limite de la API.
         * Ajustar estas extensiones y rutas a la aplicacion real.
         */
        if (isStaticResource(req)) {
            chain.doFilter(request, response);
            return;
        }

        /*
         * Origin/Referer no son autenticacion.
         * Se validan como defensa adicional cuando estan presentes
         * en peticiones que modifican estado.
         *
         * La ausencia de estos headers no bloquea integraciones
         * servidor-servidor ni clientes no browser.
         */
        if (isUnsafeMethod(req.getMethod())
                && !hasAllowedOriginIfPresent(req)) {
            reject(res, HttpServletResponse.SC_FORBIDDEN,
                    0L, "forbidden");
            return;
        }

        String key = clientKey(req);
        long now = System.currentTimeMillis();

        Window window = getOrCreateWindow(key, now);

        if (window == null) {
            maybeCleanup(now);

            reject(res, HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                    0L, "rate limiter capacity reached");
            return;
        }

        long retryAfterSec = window.tryAcquire(now);

        maybeCleanup(now);

        if (retryAfterSec > 0L) {
            reject(res, HttpServletResponse.SC_TOO_MANY_REQUESTS,
                    retryAfterSec, "too many requests");
            return;
        }

        chain.doFilter(request, response);
    }

    /**
     * Evita superar MAX_KEYS incluso si llegan peticiones
     * concurrentes para IP nuevas.
     */
    private Window getOrCreateWindow(String key, long now) {
        Window window = windows.get(key);

        if (window != null) {
            return window;
        }

        synchronized (windows) {
            window = windows.get(key);

            if (window != null) {
                return window;
            }

            if (windows.size() >= MAX_KEYS) {
                return null;
            }

            window = new Window(now);
            windows.put(key, window);
            return window;
        }
    }

    static boolean looksGarbage(HttpServletRequest req) {
        String uri = req.getRequestURI();

        if (uri == null
                || uri.length() > MAX_URI_LENGTH
                || uri.indexOf('\0') >= 0
                || uri.indexOf('\\') >= 0
                || containsDotDotSegment(uri)) {
            return true;
        }

        String query = req.getQueryString();

        return query != null
                && (query.length() > MAX_QUERY_LENGTH
                || query.indexOf('\0') >= 0);
    }

    private static boolean containsDotDotSegment(String uri) {
        String[] segments = uri.split("/", -1);

        for (int i = 0; i < segments.length; i++) {
            if ("..".equals(segments[i])) {
                return true;
            }
        }

        return false;
    }

    private static boolean isStaticResource(HttpServletRequest req) {
        String uri = req.getRequestURI();

        if (uri == null) {
            return false;
        }

        String path = uri.toLowerCase(Locale.ROOT);

        return path.endsWith(".css")
                || path.endsWith(".js")
                || path.endsWith(".png")
                || path.endsWith(".jpg")
                || path.endsWith(".jpeg")
                || path.endsWith(".gif")
                || path.endsWith(".svg")
                || path.endsWith(".ico")
                || path.endsWith(".woff")
                || path.endsWith(".woff2")
                || path.endsWith(".ttf")
                || path.endsWith(".map");
    }

    private static boolean isUnsafeMethod(String method) {
        return "POST".equalsIgnoreCase(method)
                || "PUT".equalsIgnoreCase(method)
                || "PATCH".equalsIgnoreCase(method)
                || "DELETE".equalsIgnoreCase(method);
    }

    /**
     * Si Origin existe, debe coincidir exactamente con la allowlist.
     * Si no existe, se usa Referer como señal cuando esta presente.
     * Si ambos estan ausentes, no se bloquea por este motivo.
     */
    static boolean hasAllowedOriginIfPresent(HttpServletRequest req) {
        String origin = req.getHeader("Origin");

        if (origin != null && !origin.trim().isEmpty()) {
            if ("null".equalsIgnoreCase(origin.trim())) {
                return false;
            }

            String normalized = normalizeOrigin(origin);

            return normalized != null
                    && ALLOWED_ORIGINS.contains(normalized);
        }

        String referer = req.getHeader("Referer");

        if (referer == null || referer.trim().isEmpty()) {
            return true;
        }

        String derived = originOf(referer);

        return derived != null && ALLOWED_ORIGINS.contains(derived);
    }

    static String normalizeOrigin(String raw) {
        String derived = originOf(raw);

        /*
         * Origin solo admite scheme + host + puerto.
         * No aceptar paths, query, fragmentos ni user-info.
         */
        if (derived == null) {
            return null;
        }

        try {
            URI uri = URI.create(raw.trim());

            if (uri.getRawUserInfo() != null
                    || uri.getRawPath() != null
                    && !uri.getRawPath().isEmpty()
                    || uri.getRawQuery() != null
                    || uri.getRawFragment() != null) {
                return null;
            }

            return derived;
        } catch (IllegalArgumentException ex) {
            return null;
        }
    }

    static String originOf(String value) {
        try {
            URI uri = URI.create(value.trim());

            String scheme = uri.getScheme();
            String host = uri.getHost();

            if (scheme == null || host == null
                    || uri.getRawUserInfo() != null) {
                return null;
            }

            if (!"https".equalsIgnoreCase(scheme)
                    && !"http".equalsIgnoreCase(scheme)) {
                return null;
            }

            int port = uri.getPort();

            if (port < -1 || port > 65535) {
                return null;
            }

            boolean defaultPort = port < 0
                    || ("https".equalsIgnoreCase(scheme) && port == 443)
                    || ("http".equalsIgnoreCase(scheme) && port == 80);

            String origin = scheme.toLowerCase(Locale.ROOT)
                    + "://" + host.toLowerCase(Locale.ROOT);

            return defaultPort ? origin : origin + ":" + port;

        } catch (IllegalArgumentException ex) {
            return null;
        }
    }

    static String clientKey(HttpServletRequest req) {
        String ip = req.getRemoteAddr();

        return ip == null || ip.trim().isEmpty()
                ? "unknown"
                : ip;
    }

    private void maybeCleanup(long now) {
        long previous = lastCleanup.get();

        if (now - previous < CLEANUP_EVERY_MS
                || !lastCleanup.compareAndSet(previous, now)) {
            return;
        }

        Iterator<Map.Entry<String, Window>> iterator =
                windows.entrySet().iterator();

        while (iterator.hasNext()) {
            Map.Entry<String, Window> entry = iterator.next();
            Window window = entry.getValue();

            if (window.isStale(now)) {
                /*
                 * remove(key, value) evita borrar una entrada
                 * diferente si la asociacion cambio.
                 */
                windows.remove(entry.getKey(), window);
            }
        }
    }

    private static void reject(
            HttpServletResponse res,
            int status,
            long retryAfterSec,
            String body) throws IOException {

        res.setStatus(status);

        res.setHeader("Cache-Control", "no-store");

        if (retryAfterSec > 0L) {
            res.setHeader("Retry-After",
                    Long.toString(retryAfterSec));
        }

        res.setCharacterEncoding("UTF-8");
        res.setContentType("text/plain;charset=UTF-8");
        res.getWriter().write(body);
    }

    @Override
    public void destroy() {
        windows.clear();
    }

    static final class Window {

        private long windowStart;
        private int count;
        private long bannedUntil;

        Window(long now) {
            this.windowStart = now;
        }

        synchronized long tryAcquire(long now) {
            if (now < bannedUntil) {
                return Math.max(
                        1L,
                        (bannedUntil - now + 999L) / 1000L);
            }

            if (now - windowStart >= WINDOW_MS
                    || now < windowStart) {
                windowStart = now;
                count = 0;
            }

            count++;

            if (count > MAX_REQUESTS) {
                bannedUntil = now + BAN_MS;
                count = 0;

                return BAN_MS / 1000L;
            }

            return 0L;
        }

        synchronized boolean isStale(long now) {
            return now >= bannedUntil
                    && now >= windowStart
                    && now - windowStart > WINDOW_MS * 2L;
        }
    }
}

```