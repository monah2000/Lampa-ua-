# Lampa-ua-(function() {
    'use strict';

    // --- НАЛАШТУВАННЯ ---
    const PLUGIN_ID = 'universal_hd_sources';
    const PLUGIN_VERSION = '1.0.0';
    // Посилання на файл, де зберігатиметься актуальний список джерел
    const SOURCES_LIST_URL = 'https://raw.githubusercontent.com/your-username/your-repo/main/sources_list.json';
    const UPDATE_CHECK_INTERVAL = 24 * 60 * 60 * 1000; // Перевірка оновлень раз на добу
    // ----------------------

    // Запобігання повторній ініціалізації
    if (window[PLUGIN_ID + '_loaded']) {
        Lampa.Noty.show('Плагін ' + PLUGIN_ID + ' вже завантажено.', {type: 'info', timeout: 2000});
        return;
    }
    window[PLUGIN_ID + '_loaded'] = true;

    // Глобальний об'єкт плагіна для зберігання даних
    const plugin = {
        sources: [], // Тут зберігатиметься завантажений список джерел
        lastUpdateCheck: 0
    };

    // Допоміжна функція для логування
    function log(message, data) {
        console.log(`[${PLUGIN_ID}] ${message}`, data || '');
    }

    // Допоміжна функція для відображення повідомлень
    function notify(message, type = 'info', timeout = 3000) {
        Lampa.Noty.show(message, {type: type, timeout: timeout});
    }

    // Функція для завантаження списку джерел
    function loadSourcesList() {
        return new Promise((resolve, reject) => {
            Lampa.Network.get(SOURCES_LIST_URL, (data) => {
                try {
                    const parsedData = JSON.parse(data);
                    if (parsedData && parsedData.sources && Array.isArray(parsedData.sources)) {
                        plugin.sources = parsedData.sources;
                        Lampa.Storage.set(PLUGIN_ID + '_sources', plugin.sources);
                        Lampa.Storage.set(PLUGIN_ID + '_last_update', Date.now());
                        log('Список джерел успішно завантажено та збережено.');
                        notify('Список джерел оновлено!', 'success');
                        resolve(plugin.sources);
                    } else {
                        throw new Error('Невірний формат даних у списку джерел');
                    }
                } catch (e) {
                    log('Помилка при парсингу списку джерел:', e);
                    reject(e);
                }
            }, (error) => {
                log('Помилка мережі при завантаженні списку джерел:', error);
                reject(error);
            });
        });
    }

    // Функція для перевірки оновлень списку джерел
    function checkForUpdates(force = false) {
        const now = Date.now();
        const lastCheck = Lampa.Storage.get(PLUGIN_ID + '_last_update', 0);
        const timeSinceLastCheck = now - lastCheck;

        if (!force && timeSinceLastCheck < UPDATE_CHECK_INTERVAL) {
            log(`Перевірку оновлень пропущено (минуло ${Math.round(timeSinceLastCheck / 3600000)} год.)`);
            return;
        }

        log('Перевірка оновлень списку джерел...');
        loadSourcesList()
            .then(() => {
                log('Список джерел оновлено.');
            })
            .catch(error => {
                log('Не вдалося оновити список джерел:', error);
                // Якщо оновлення не вдалося, пробуємо завантажити збережений список
                const savedSources = Lampa.Storage.get(PLUGIN_ID + '_sources');
                if (savedSources && savedSources.length > 0) {
                    plugin.sources = savedSources;
                    log('Використовується збережений список джерел.');
                } else {
                    notify('Не вдалося завантажити джерела. Перевірте з\'єднання.', 'error');
                }
            });
    }

    // Функція для створення пункту меню з джерелами
    function createSourcesMenu() {
        const menuItems = plugin.sources.map(source => ({
            title: source.name,
            description: source.description || `${source.type} джерело`,
            icon: source.icon || 'folder',
            action: () => {
                // Відкриваємо каталог з вибраним джерелом
                Lampa.Activity.push({
                    url: source.url,
                    title: source.name,
                    component: source.component || 'category',
                    page: 1
                });
            }
        }));

        // Додаємо пункт "Оновити джерела"
        menuItems.push({
            title: '🔄 Оновити список джерел',
            description: 'Завантажити актуальний список з сервера',
            icon: 'refresh',
            action: () => {
                notify('Оновлення списку джерел...', 'info');
                checkForUpdates(true);
            }
        });

        // Додаємо пункт "Про плагін"
        menuItems.push({
            title: 'ℹ️ Про плагін',
            description: `Версія ${PLUGIN_VERSION}`,
            icon: 'info',
            action: () => {
                Lampa.Activity.push({
                    url: '',
                    title: 'Про плагін',
                    component: 'text',
                    text: `
                        <h2>Універсальний плагін джерел FullHD/4K</h2>
                        <p>Версія: ${PLUGIN_VERSION}</p>
                        <p>Плагін автоматично завантажує та оновлює список джерел контенту у високій якості.</p>
                        <p>Список джерел оновлюється раз на добу. Ви також можете оновити його вручну через меню.</p>
                        <p>Для додавання нових джерел відредагуйте файл <code>sources_list.json</code> за посиланням:</p>
                        <pre>${SOURCES_LIST_URL}</pre>
                    `
                });
            }
        });

        return menuItems;
    }

    // Основна функція ініціалізації плагіна
    function init() {
        log('Ініціалізація плагіна...');

        // Завантажуємо збережений список джерел
        const savedSources = Lampa.Storage.get(PLUGIN_ID + '_sources');
        if (savedSources && savedSources.length > 0) {
            plugin.sources = savedSources;
            log('Завантажено збережений список джерел.');
        } else {
            // Якщо немає збереженого, завантажуємо з мережі
            loadSourcesList().catch(() => {
                notify('Не вдалося завантажити джерела. Перевірте з\'єднання.', 'error');
            });
        }

        // Додаємо пункт у головне меню Lampa
        Lampa.Listener.follow('menu', (e) => {
            if (e.type === 'ready') {
                // Видаляємо старий пункт, якщо він є
                const existingIndex = e.menu.findIndex(item => item.id === PLUGIN_ID);
                if (existingIndex !== -1) {
                    e.menu.splice(existingIndex, 1);
                }

                // Додаємо новий пункт меню
                e.menu.push({
                    id: PLUGIN_ID,
                    title: '🎬 FullHD Джерела',
                    description: 'Каталоги фільмів та серіалів у високій якості',
                    icon: 'movie',
                    items: createSourcesMenu()
                });
                log('Пункт меню додано.');
            }
        });

        // Плануємо перевірку оновлень
        setTimeout(() => {
            checkForUpdates();
        }, 5000); // Затримка 5 секунд після запуску

        // Реєструємо плагін в системі Lampa
        Lampa.Plugin.register({
            id: PLUGIN_ID,
            name: 'Універсальний плагін FullHD джерел',
            description: 'Автоматично оновлюваний каталог джерел контенту у високій якості',
            version: PLUGIN_VERSION,
            start: init
        });

        notify('Плагін FullHD джерел активовано!', 'success');
        log('Плагін успішно ініціалізовано.');
    }

    // Запускаємо ініціалізацію, коли Lampa готова
    if (window.Lampa) {
        // Lampa вже завантажена
        init();
    } else {
        // Чекаємо на завантаження Lampa
        window.addEventListener('load', () => {
            if (window.Lampa) {
                init();
            } else {
                log('Помилка: Lampa не знайдено.');
            }
        });
    }

})();{
  "sources": [
    {
      "name": "LNUM (Каталог)",
      "type": "catalog",
      "url": "line%3Dbase%26id%3Dlnum%26page%3D1",
      "component": "category",
      "description": "Популярний каталог фільмів та серіалів"
    },
    {
      "name": "Толока (Торент)",
      "type": "torrent",
      "url": "line%3Dtorrent%26id%3Dtoloka%26page%3D1",
      "component": "category",
      "description": "Пошук торентів на Toloka"
    },
    {
      "name": "UAKino",
      "type": "online",
      "url": "line%3Donline%26id%3Duakino%26page%3D1",
      "component": "category",
      "description": "Український онлайн-кінотеатр"
    },
    {
      "name": "HDRezka",
      "type": "online",
      "url": "line%3Donline%26id%3Dhdrezka%26page%3D1",
      "component": "category",
      "description": "Популярний онлайн-кінотеатр з великим вибором"
    }
    // Додавайте сюди інші джерела
  ]
}
