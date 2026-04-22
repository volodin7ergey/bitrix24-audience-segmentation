<h1 align="center">Автоматизация формирования клиентской базы для email-рассылок 📬</h1>

<p align="center">
ETL-инструмент для выгрузки контактов из CRM, обработки данных и отправки в сервис email-рассылок
</p>

<p align="center">
<b>🔄 Автоматизация выгрузки • 🧹 Очистка данных • 📤 Контролируемая отправка</b>
</p>

<hr>

<h2>🎥 Демонстрация</h2>

<div align="center">
<img src="media/demo.gif" width="100%">
<br>
<em>Процесс загрузки, проверки и отправки контактов</em>
</div>

<br>

<table width="100%">
<tr>
<td align="center">
<img src="media/screen-1.jpg" width="100%">
<br>
  
<em>Начальный экран</em>
</td>

<td align="center">
<img src="media/screen-2.jpg" width="100%">
  
<br>

<em>Вывод данных</em>
</td>
</tr>

<tr>
<td align="center">
  
<img src="media/screen-3.jpg" width="100%">

<br>

<em>Проверка данных перед отправкой</em>
</td>

<td align="center">
  
<img src="media/screen-4.jpg" width="100%">
  
<br>

<em>Успешная отправка в Unisender</em>
</td>
</tr>
</table>

<hr>

<h2>🧩 Контекст задачи</h2>

<p>
Клиент использовал Bitrix24 как основную CRM, а Unisender — для email-рассылок.
Возникла задача автоматизировать передачу контактов между системами с учетом бизнес-логики.
</p>

<p>
Важно было не просто выгружать данные, а:
</p>

<ul>
<li>учитывать только релевантные сделки</li>
<li>фильтровать некорректные контакты</li>
<li>исключить риск массовых ошибок при отправке</li>
</ul>

<hr>

<h2>💡 Что было реализовано</h2>

<ul>
<li>Массовая загрузка сделок, контактов и компаний из Bitrix24</li>
<li>Фильтрация контактов без email и без привязки к сделке</li>
<li>Поддержка кастомных полей (город, оборудование, выставка и др.)</li>

<li><b>Двухэтапная отправка данных (Dry-run + реальная отправка)</b></li>

<li>Dry-run режим для предварительной проверки данных</li>
<li>Отправка данных в Unisender через API</li>
<li>Отображение прогресса отправки в реальном времени</li>
<li>Подтверждение успешной отправки</li>
</ul>

<hr>

<h2>⚙️ Логика работы</h2>

<ul>
<li>Загружаются сделки по выбранным направлениям</li>
<li>Определяются связанные контакты и компании</li>
<li>Фильтруются контакты без email</li>
<li>Данные очищаются от пустых значений</li>
<li>Формируется база контактов для Unisender</li>
<li>Происходит двухэтапная проверка данных перед отправкой</li>
<li>Клиентская база отправляется в Unisender</li>
</ul>

<h3>📤 Двухэтапная отправка</h3>

<ul>
<li><b>Шаг 1 — Dry-run (проверка данных)</b></li>
<li><b>Шаг 2 — Реальная отправка в Unisender</b></li>
</ul>

<p>
Перед реальной отправкой пользователь сначала запускает режим проверки (dry-run).
На этом этапе система имитирует отправку и показывает:
</p>

<ul>
<li>общее количество контактов</li>
<li>прогресс обработки</li>
<li>готовность данных к отправке</li>
</ul>

<p>
Это позволяет убедиться, что:
</p>

<ul>
<li>данные корректно собраны</li>
<li>нет пустых или некорректных значений</li>
<li>логика фильтрации работает правильно</li>
</ul>

<p>
Только после этого пользователь вручную подтверждает реальную отправку.
</p>

<p>
<b>Зачем это нужно:</b>
</p>

<ul>
<li>❌ исключает риск случайной отправки некорректных данных</li>
<li>📉 снижает вероятность ошибок при массовой загрузке</li>
<li>🔍 дает прозрачность — пользователь видит, что именно будет отправлено</li>
<li>🛡 защищает базу Unisender от "засорения"</li>
</ul>

<hr>

<h2>📊 Особенности бизнес-логики</h2>

<table>
<tr>
<th>Правило</th>
<th>Описание</th>
</tr>

<tr>
<td>Фильтр email</td>
<td>Контакт не отправляется без email</td>
</tr>

<tr>
<td>Связь со сделкой</td>
<td>Контакт должен быть привязан к сделке</td>
</tr>

<tr>
<td>Направления</td>
<td>Учитываются только выбранные воронки</td>
</tr>

<tr>
<td>Agreement</td>
<td>Рассчитывается по стадии сделки и воронке</td>
</tr>
</table>

<br>

<ul>
<li>🟢 Согласие = true → контакт можно использовать в рассылке</li>
<li>🔴 Согласие = false → контакт остается, но помечен</li>
<li>📤 Все данные отправляются в единый список Unisender</li>
</ul>

<hr>

<h2>🛠 Технологический стек</h2>

<table width="100%" cellpadding="10">
<tr>
<td align="center">
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/vuejs/vuejs-original.svg" width="40"><br>
<b>Vue 3</b>
</td>

<td align="center">
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/vuetify/vuetify-original.svg" width="40"><br>
<b>Vuetify</b>
</td>

<td align="center">
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/typescript/typescript-original.svg" width="40"><br>
<b>TypeScript</b>
</td>

<td align="center">
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/vitejs/vitejs-original.svg" width="40"><br>
<b>Vite</b>
</td>

<td align="center">
<img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/javascript/javascript-original.svg" width="40"><br>
<b>Fetch API</b>
</td>

<td align="center">
<img src="https://github.com/user-attachments/assets/8d146d5c-1c3e-40e5-af64-584c1aadf25f" width="40"><br>
<b>Bitrix24 REST API</b>
</td>

<td align="center">
<img src="https://www.unisender.com/favicon.ico" width="40"><br>
<b>Unisender API</b>
</td>
</tr>
</table>

<hr>

<h2>📩 Контакты</h2>

<p>
Telegram: <a href="https://t.me/volodin7ergey">@volodin7ergey</a><br>
VK: <a href="https://vk.com/volodin7ergey">vk.com/volodin7ergey</a>
</p>

<p align="center">
<b>Готов разработать аналогичные решения под Ваши бизнес-процессы 💼</b>
</p>
