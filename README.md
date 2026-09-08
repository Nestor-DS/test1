```
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Test Google reCAPTCHA Site Key</title>
    <style>
        :root { color-scheme: dark; }
        body {
            font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
            background: #0b0d10;
            color: #e6edf3;
            max-width: 720px;
            margin: 40px auto;
            padding: 0 16px;
            line-height: 1.5;
        }
        h1 { font-size: 1.25rem; margin-bottom: 0.25rem; }
        .muted { color: #8b949e; font-size: 0.9rem; }
        label { display: block; margin: 16px 0 6px; font-weight: 600; }
        input[type="text"] {
            width: 100%;
            box-sizing: border-box;
            padding: 10px 12px;
            background: #161b22;
            border: 1px solid #30363d;
            color: #e6edf3;
            border-radius: 6px;
            font: inherit;
        }
        button {
            margin-top: 12px;
            padding: 10px 16px;
            background: #238636;
            color: #fff;
            border: 0;
            border-radius: 6px;
            font: inherit;
            font-weight: 700;
            cursor: pointer;
        }
        button:hover { background: #2ea043; }
        #status { margin-top: 16px; white-space: pre-wrap; }
        .ok { color: #3fb950; }
        .bad { color: #f85149; }
        #widget { margin-top: 24px; min-height: 80px; }
        code { background: #161b22; padding: 2px 6px; border-radius: 4px; }
    </style>
</head>
<body>
    <h1>Google reCAPTCHA v2 — site key smoke test</h1>
    <p class="muted">
        Esto NO verifica el secret key. El secret vive en el backend.
        Aquí solo pruebas si el <strong>SITE KEY</strong> carga el widget y genera un token.
        El dominio tiene que estar en la consola de reCAPTCHA o usa localhost.
    </p>

    <label for="sitekey">SITE KEY (pública)</label>
    <input id="sitekey" type="text" autocomplete="off" spellcheck="false"
           placeholder="6Le...  — no pegues el SECRET aquí, idiota">

    <button type="button" id="loadBtn">Cargar widget</button>
    <button type="button" id="tokenBtn" disabled>Obtener token</button>

    <p id="status" class="muted">Pega la site key y carga. Google test key que siempre pasa:
        <code>6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI</code>
    </p>

    <div id="widget"></div>

    <script src="https://www.google.com/recaptcha/api.js?render=explicit" async defer></script>
    <script>
        (function () {
            const sitekeyEl = document.getElementById('sitekey');
            const statusEl = document.getElementById('status');
            const widgetEl = document.getElementById('widget');
            const loadBtn = document.getElementById('loadBtn');
            const tokenBtn = document.getElementById('tokenBtn');
            let widgetId = null;

            function setStatus(msg, cls) {
                statusEl.className = cls || 'muted';
                statusEl.textContent = msg;
            }

            function waitGrecaptcha() {
                return new Promise(function (resolve, reject) {
                    let n = 0;
                    (function tick() {
                        if (window.grecaptcha && typeof grecaptcha.render === 'function') {
                            resolve();
                            return;
                        }
                        if (++n > 80) {
                            reject(new Error('grecaptcha no cargó. Red, adblock, o Google se fue a la mierda.'));
                            return;
                        }
                        setTimeout(tick, 50);
                    })();
                });
            }

            loadBtn.addEventListener('click', async function () {
                const key = sitekeyEl.value.trim();
                if (!key) {
                    setStatus('NAK. No hay site key. ¿Qué mierda esperabas que pasara?', 'bad');
                    return;
                }
                if (key.length < 20) {
                    setStatus('NAK. Eso no parece una site key de Google. Deja de pegar basura.', 'bad');
                    return;
                }

                widgetEl.innerHTML = '';
                widgetId = null;
                tokenBtn.disabled = true;
                setStatus('Cargando widget...', 'muted');

                try {
                    await waitGrecaptcha();
                    widgetId = grecaptcha.render(widgetEl, {
                        sitekey: key,
                        theme: 'dark',
                        callback: function () {
                            setStatus('Checkbox OK. Ahora pide el token.', 'ok');
                            tokenBtn.disabled = false;
                        },
                        'expired-callback': function () {
                            setStatus('Token expiró. Vuelve a marcar la caja.', 'bad');
                            tokenBtn.disabled = true;
                        },
                        'error-callback': function () {
                            setStatus('ERROR de Google. Site key inválida, dominio no autorizado, o quota. Revisa la consola de reCAPTCHA.', 'bad');
                            tokenBtn.disabled = true;
                        }
                    });
                    setStatus('Widget renderizado. Marca "No soy un robot". Si no aparece, la key o el dominio están mal.', 'muted');
                } catch (err) {
                    setStatus('FALLO: ' + err.message, 'bad');
                }
            });

            tokenBtn.addEventListener('click', function () {
                if (widgetId === null) return;
                const token = grecaptcha.getResponse(widgetId);
                if (!token) {
                    setStatus('No hay token. Marca el captcha primero, cabrón.', 'bad');
                    return;
                }
                setStatus('TOKEN (máx ~2 min de vida):\n' + token, 'ok');
            });
        })();
    </script>
</body>
</html>
```