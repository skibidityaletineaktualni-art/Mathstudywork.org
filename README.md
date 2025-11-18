<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>BoardyLordy</title>
<style>
    body { background:#000; color:#00ff66; font-family:Consolas, monospace; margin:0; }
    header { text-align:center; padding:20px; font-size:40px; font-weight:bold; border-bottom:2px solid #00ff66; }
    .button { padding:15px; margin:10px; border:2px solid #00ff66; border-radius:8px; cursor:pointer; text-align:center; }
    .button:hover { background:#003311; }
    #game-list { padding:20px; max-width:600px; margin:auto; }
    #game-frame { width:100%; height:600px; border:none; display:none; margin-top:20px; }
    #back { display:none; padding:10px; margin:20px auto; width:200px; text-align:center; border:2px solid #00ff66; cursor:pointer; }
    #back:hover { background:#003311; }

    #addMenu {
        display:none;
        padding:20px;
        max-width:600px;
        margin:30px auto;
        border:2px solid #00ff66;
        border-radius:10px;
    }
    textarea {
        width:100%;
        height:300px;
        background:black;
        color:#0f0;
        border:2px solid #00ff66;
        font-family:Consolas;
        padding:10px;
    }
    input {
        width:100%;
        padding:10px;
        background:black;
        color:#0f0;
        border:2px solid #00ff66;
        margin-bottom:10px;
        font-family:Consolas;
    }
</style>
</head>

<body>

<header>BoardyLordy</header>

<div id="game-list">
    <h2>Игры:</h2>
    <div id="games"></div>

    <div class="button" onclick="openAddGame()">Add Game</div>
</div>

<!-- Меню добавления игры -->
<div id="addMenu">
    <h2>Добавить игру</h2>
    <input type="text" id="gameName" placeholder="Название игры">
    <textarea id="gameCode" placeholder="Вставь сюда HTML игры"></textarea>
    <div class="button" onclick="saveGame()">Добавить</div>
    <div class="button" onclick="closeAddMenu()">Отмена</div>
</div>

<div id="back" onclick="backToMenu()">← Назад</div>
<iframe id="game-frame"></iframe>

<script>
// Загружаем игры из localStorage
let games = JSON.parse(localStorage.getItem("games"))  {};

function renderGames() {
    const container = document.getElementById("games");
    container.innerHTML = "";

    Object.keys(games).forEach(name => {
        let btn = document.createElement("div");
        btn.className = "button";
        btn.textContent = name;
        btn.onclick = () => loadGame(name);
        container.appendChild(btn);
    });
}
renderGames();

// Загрузка игры
function loadGame(name) {
    const frame = document.getElementById("game-frame");
    frame.srcdoc = games[name];
    frame.style.display = "block";
    document.getElementById("game-list").style.display = "none";
    document.getElementById("back").style.display = "block";
}

function backToMenu() {
    document.getElementById("game-frame").style.display = "none";
    document.getElementById("game-list").style.display = "block";
    document.getElementById("back").style.display = "none";
}

// Открытие меню добавления
function openAddGame() {
    let pass = prompt("Введите пароль:");
    if (pass !== "lemontap1") {
        alert("Неправильный пароль.");
        return;
    }
    document.getElementById("addMenu").style.display = "block";
    document.getElementById("game-list").style.display = "none";
}

// Закрыть меню
function closeAddMenu() {
    document.getElementById("addMenu").style.display = "none";
    document.getElementById("game-list").style.display = "block";
}

// Сохранение игры
function saveGame() {
    const name = document.getElementById("gameName").value.trim();
    const code = document.getElementById("gameCode").value.trim();

    if (!name  !code) {
        alert("Заполни название и код игры!");
        return;
    }

    games[name] = code;
    localStorage.setItem("games", JSON.stringify(games));

    renderGames();
    closeAddMenu();
    alert("Игра добавлена!");
}
</script>

</body>
</html>
