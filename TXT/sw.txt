// === PDF Tools Suite - Service Worker ===
// Versión de la caché. Al cambiar este número, el SW descarta la caché vieja
// y descarga todos los archivos frescos en la próxima visita.
const CACHE_NAME = 'pdf-tools-v4';

// Lista de todos los archivos a cachear para uso offline
const ASSETS_TO_CACHE = [
    './',
    './index.html',
    './styles.css',
    './theme.js',
    './manifest.json',
    './icon-192.png',
    './icon-512.png',
    // Librerías
    './pdf-lib.min.js',
    './pdf.min.js',
    './pdf.worker.min.js',
    './jszip.min.js',
    // Herramientas
    './UnirPDF.html',
    './DividirPDF.html',
    './GirarPDF.html',
    './ImgPDF.html',
    './PDFImg.html',
    './EliminarPDF.html',
    './OrdenarPDF.html',
    './ExtraerTexto.html',
    './ComprimirPDF.html',
    './MarcaAgua.html',
    './NumerarPDF.html',
];

// --- INSTALL: Pre-cachear todos los archivos estáticos ---
self.addEventListener('install', (event) => {
    event.waitUntil(
        caches.open(CACHE_NAME).then((cache) => {
            console.log('[SW] Pre-cacheando archivos de la suite...');
            return cache.addAll(ASSETS_TO_CACHE);
        }).then(() => {
            // Forzar activación inmediata sin esperar a que se cierren las otras pestañas
            return self.skipWaiting();
        })
    );
});

// --- ACTIVATE: Limpiar cachés antiguas ---
self.addEventListener('activate', (event) => {
    event.waitUntil(
        caches.keys().then((cacheNames) => {
            return Promise.all(
                cacheNames
                    .filter((name) => name !== CACHE_NAME)
                    .map((name) => {
                        console.log('[SW] Eliminando caché antigua:', name);
                        return caches.delete(name);
                    })
            );
        }).then(() => self.clients.claim())
    );
});

// --- FETCH: Estrategia Cache-First (offline primero) ---
// 1. Busca el recurso en la caché.
// 2. Si está en caché, lo sirve al instante (sin red).
// 3. Si NO está en caché (ej: un archivo nuevo), lo descarga de la red y lo guarda para la próxima.
self.addEventListener('fetch', (event) => {
    // Solo interceptar peticiones GET (no POST, etc.)
    if (event.request.method !== 'GET') return;

    event.respondWith(
        caches.match(event.request).then((cachedResponse) => {
            if (cachedResponse) {
                return cachedResponse;
            }
            // No está en caché: buscar en red y guardar
            return fetch(event.request).then((networkResponse) => {
                // Solo cachear respuestas válidas
                if (!networkResponse || networkResponse.status !== 200 || networkResponse.type === 'opaque') {
                    return networkResponse;
                }
                const responseToCache = networkResponse.clone();
                caches.open(CACHE_NAME).then((cache) => {
                    cache.put(event.request, responseToCache);
                });
                return networkResponse;
            }).catch(() => {
                // Sin red y sin caché: mostrar página de error offline si existe
                return caches.match('./index.html');
            });
        })
    );
});

// Asegurar que el SW se actualiza correctamente
self.addEventListener('message', (event) => {
    if (event.data && event.data.type === 'SKIP_WAITING') {
        self.skipWaiting();
    }
});
