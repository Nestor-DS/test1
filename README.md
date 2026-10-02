```

package mx.com.inbursa.quimera.core.service;

import java.io.IOException;
import java.util.Collections;
import java.util.HashSet;
import java.util.Iterator;
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
 * Solo paths vigilados: 20 en la ventana, ban 2 min.
 * Un JVM. La clave es IP + path. Otro path no suma y no se banea.
 */
public class AbuseGateFilter implements Filter {

    static final int MAX_REQUESTS = 20;
    static final long WINDOW_MS = 60_000L;
    static final long BAN_MS = 2 * 60_000L;
    static final int MAX_KEYS = 100_000;
    static final long CLEANUP_EVERY_MS = 60_000L;

    /** Path sin context path, sin query, sin slash final. Exacto. */
    static final Set<String> WATCHED;
    static {
        Set<String> paths = new HashSet<String>();
        paths.add("/quimera/pago");
        paths.add("/quimera/transferencia");
        paths.add("/quimera/token");
        WATCHED = Collections.unmodifiableSet(paths);
    }

    private final ConcurrentHashMap<String, Window> windows = new ConcurrentHashMap<String, Window>();
    private final AtomicLong lastCleanup = new AtomicLong(System.currentTimeMillis());

    @Override
    public void init(FilterConfig filterConfig) {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        if (!(request instanceof HttpServletRequest) || !(response instanceof HttpServletResponse)) {
            chain.doFilter(request, response);
            return;
        }

        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse res = (HttpServletResponse) response;

        String path = watchedPath(req);
        if (path == null) {
            chain.doFilter(req, res);
            return;
        }

        String key = clientKey(req) + " " + path;
        long now = System.currentTimeMillis();

        if (windows.size() >= MAX_KEYS && !windows.containsKey(key)) {
            reject(res, BAN_MS / 1000L);
            return;
        }

        Window window = windowFor(key, now);
        long retryAfterSec = window.tryAcquire(now);
        maybeCleanup(now);

        if (retryAfterSec > 0L) {
            reject(res, retryAfterSec);
            return;
        }

        chain.doFilter(req, res);
    }

    @Override
    public void destroy() {
        windows.clear();
    }

    /**
     * null = este request no se cuenta ni se banea.
     * Compara servletPath + pathInfo, no el URI crudo con context path duplicado.
     */
    static String watchedPath(HttpServletRequest req) {
        String servletPath = req.getServletPath();
        String pathInfo = req.getPathInfo();
        String raw = (servletPath == null ? "" : servletPath) + (pathInfo == null ? "" : pathInfo);
        if (raw.length() == 0) {
            raw = req.getRequestURI();
            String context = req.getContextPath();
            if (raw != null && context != null && context.length() > 0 && raw.startsWith(context)) {
                raw = raw.substring(context.length());
            }
        }
        if (raw == null || raw.indexOf('\0') >= 0 || raw.indexOf("..") >= 0) {
            return null;
        }
        if (raw.length() > 1 && raw.endsWith("/")) {
            raw = raw.substring(0, raw.length() - 1);
        }
        return WATCHED.contains(raw) ? raw : null;
    }

    private Window windowFor(String key, long now) {
        Window window = windows.get(key);
        if (window != null) {
            return window;
        }
        Window fresh = new Window(now);
        Window prev = windows.putIfAbsent(key, fresh);
        return prev == null ? fresh : prev;
    }

    static String clientKey(HttpServletRequest req) {
        String ip = req.getRemoteAddr();
        return ip == null || ip.length() == 0 ? "unknown" : ip;
    }

    private void maybeCleanup(long now) {
        long prev = lastCleanup.get();
        if (now - prev < CLEANUP_EVERY_MS || !lastCleanup.compareAndSet(prev, now)) {
            return;
        }
        Iterator<Map.Entry<String, Window>> it = windows.entrySet().iterator();
        while (it.hasNext()) {
            if (it.next().getValue().isStale(now)) {
                it.remove();
            }
        }
    }

    private static void reject(HttpServletResponse res, long retryAfterSec) throws IOException {
        res.setStatus(429);
        res.setHeader("Retry-After", Long.toString(retryAfterSec));
        res.setCharacterEncoding("UTF-8");
        res.setContentType("text/plain;charset=UTF-8");
        res.getWriter().write("too many requests");
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
                return Math.max(1L, (bannedUntil - now + 999L) / 1000L);
            }
            if (now - windowStart >= WINDOW_MS) {
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
            return now >= bannedUntil && now - windowStart > WINDOW_MS * 2L;
        }
    }
}
```