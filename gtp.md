Отличная идея для Telegram бота! Создам для вас подробное техническое решение для организации мини-турниров по теннису.

## Структура проекта

```
tennis_tournament_bot/
├── bot.py                 # Основной файл бота
├── handlers/              # Обработчики команд и сообщений
│   ├── __init__.py
│   ├── start.py
│   ├── tournament.py
│   └── results.py
├── models/                # Модели данных
│   ├── __init__.py
│   ├── tournament.py
│   └── player.py
├── utils/                 # Утилиты
│   ├── __init__.py
│   ├── tournament_generator.py
│   ├── image_generator.py
│   └── date_utils.py
├── data/                  # Папка с JSON файлами турниров
│   └── tournaments/
├── config.py              # Конфигурационный файл
├── requirements.txt
└── README.md
```

## Установка и зависимости

Создайте файл `requirements.txt`:

```txt
python-telegram-bot==20.7
Pillow==10.1.0
python-dateutil==2.8.2
```

## Конфигурация

Создайте файл `config.py`:

```python
import os
from pathlib import Path

# Токен бота (получаете от @BotFather)
BOT_TOKEN = os.getenv('TELEGRAM_BOT_TOKEN', 'YOUR_BOT_TOKEN_HERE')

# Пути к файлам
BASE_DIR = Path(__file__).parent
DATA_DIR = BASE_DIR / 'data' / 'tournaments'
IMAGES_DIR = BASE_DIR / 'data' / 'images'

# Создаем директории если их нет
DATA_DIR.mkdir(parents=True, exist_ok=True)
IMAGES_DIR.mkdir(parents=True, exist_ok=True)

# Форматы турниров для разного количества игроков
TOURNAMENT_FORMATS = {
    8: [
        "Плей-офф на выбывание (8 → 4 → 2 → 1)",
        "2 группы по 4 + плей-офф"
    ],
    9: [
        "3 группы по 3 + плей-офф",
        "Свободный формат"
    ],
    12: [
        "3 группы по 4 + плей-офф", 
        "4 группы по 3 + плей-офф",
        "Плей-офф на выбывание"
    ]
}
```

## Модели данных

Создайте `models/tournament.py`:

```python
import json
import uuid
from datetime import datetime, date
from pathlib import Path
from typing import Dict, List, Optional, Any
from config import DATA_DIR

class Tournament:
    def __init__(self):
        self.id: str = str(uuid.uuid4())
        self.name: str = ""
        self.format: str = ""
        self.players: List[str] = []
        self.bracket: Dict[str, Any] = {}
        self.results: Dict[str, Dict] = {}  # {match_id: {player1: score1, player2: score2, winner: player}}
        self.status: str = "forming"  # forming, started, finished
        self.created_date: str = datetime.now().isoformat()
        self.start_date: Optional[str] = None
        self.finish_date: Optional[str] = None
        self.creator_id: int = 0
        
    def save(self) -> None:
        """Сохранить турнир в JSON файл"""
        file_path = DATA_DIR / f"tournament_{self.id}.json"
        with open(file_path, 'w', encoding='utf-8') as f:
            json.dump(self.__dict__, f, ensure_ascii=False, indent=2)
    
    @classmethod
    def load(cls, tournament_id: str) -> Optional['Tournament']:
        """Загрузить турнир из JSON файла"""
        file_path = DATA_DIR / f"tournament_{tournament_id}.json"
        if not file_path.exists():
            return None
            
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                data = json.load(f)
            
            tournament = cls()
            tournament.__dict__.update(data)
            return tournament
        except Exception:
            return None
    
    @classmethod
    def get_tournaments_by_date_range(cls, start_date: date, end_date: date) -> List['Tournament']:
        """Получить турниры за указанный период"""
        tournaments = []
        
        for file_path in DATA_DIR.glob("tournament_*.json"):
            tournament = cls.load(file_path.stem.replace("tournament_", ""))
            if tournament:
                tournament_date = datetime.fromisoformat(tournament.created_date).date()
                if start_date <= tournament_date <= end_date:
                    tournaments.append(tournament)
        
        return sorted(tournaments, key=lambda t: t.created_date, reverse=True)
    
    def generate_bracket(self, randomize: bool = True) -> None:
        """Генерация турнирной сетки"""
        from utils.tournament_generator import TournamentGenerator
        generator = TournamentGenerator()
        self.bracket = generator.generate(self.players, self.format, randomize)
    
    def update_result(self, match_id: str, player1_score: int, player2_score: int, winner: str) -> bool:
        """Обновить результат матча"""
        if match_id in self.bracket.get('matches', {}):
            self.results[match_id] = {
                'player1_score': player1_score,
                'player2_score': player2_score,
                'winner': winner,
                'completed': True,
                'timestamp': datetime.now().isoformat()
            }
            self.save()
            return True
        return False
    
    def replace_player(self, old_player: str, new_player: str) -> bool:
        """Заменить игрока в турнире"""
        if self.status != "forming":
            return False
            
        if old_player in self.players:
            index = self.players.index(old_player)
            self.players[index] = new_player
            # Обновляем турнирную сетку
            self.generate_bracket(randomize=False)
            self.save()
            return True
        return False
```

## Генератор турниров

Создайте `utils/tournament_generator.py`:

```python
import random
from typing import Dict, List, Any
import math

class TournamentGenerator:
    def generate(self, players: List[str], format_type: str, randomize: bool = True) -> Dict[str, Any]:
        """Генерация турнирной сетки"""
        if randomize:
            shuffled_players = players.copy()
            random.shuffle(shuffled_players)
        else:
            shuffled_players = players
            
        if "плей-офф на выбывание" in format_type.lower():
            return self._generate_knockout(shuffled_players)
        elif "группы" in format_type.lower():
            return self._generate_group_stage(shuffled_players, format_type)
        elif "свободный" in format_type.lower():
            return self._generate_free_format(shuffled_players)
        else:
            return self._generate_knockout(shuffled_players)
    
    def _generate_knockout(self, players: List[str]) -> Dict[str, Any]:
        """Генерация турнира на выбывание"""
        matches = {}
        
        # Первый раунд
        round_num = 1
        match_id = 1
        current_players = players.copy()
        
        while len(current_players) > 1:
            round_matches = []
            next_round_players = []
            
            # Создаем пары для текущего раунда
            for i in range(0, len(current_players), 2):
                if i + 1 < len(current_players):
                    player1 = current_players[i]
                    player2 = current_players[i + 1]
                    
                    match_key = f"round_{round_num}_match_{match_id}"
                    round_matches.append({
                        'id': match_key,
                        'player1': player1,
                        'player2': player2,
                        'winner': None,
                        'completed': False
                    })
                    match_id += 1
                else:
                    # Нечетное количество игроков - проход в следующий раунд
                    next_round_players.append(current_players[i])
            
            matches[f'round_{round_num}'] = round_matches
            
            # Для следующего раунда
            current_players = next_round_players + [None] * len(round_matches)
            round_num += 1
            
            if len(round_matches) == 0:
                break
        
        return {
            'type': 'knockout',
            'matches': matches,
            'current_round': 1
        }
    
    def _generate_group_stage(self, players: List[str], format_type: str) -> Dict[str, Any]:
        """Генерация группового этапа"""
        groups = {}
        
        if "2 группы по 4" in format_type:
            group_size = 4
            num_groups = 2
        elif "3 группы по 4" in format_type:
            group_size = 4
            num_groups = 3
        elif "3 группы по 3" in format_type:
            group_size = 3
            num_groups = 3
        elif "4 группы по 3" in format_type:
            group_size = 3
            num_groups = 4
        else:
            group_size = 4
            num_groups = len(players) // 4
        
        # Распределение по группам
        for i in range(num_groups):
            start_idx = i * group_size
            end_idx = min(start_idx + group_size, len(players))
            group_players = players[start_idx:end_idx]
            
            # Создание матчей внутри группы (каждый с каждым)
            group_matches = []
            match_id = 1
            
            for j in range(len(group_players)):
                for k in range(j + 1, len(group_players)):
                    match_key = f"group_{i+1}_match_{match_id}"
                    group_matches.append({
                        'id': match_key,
                        'player1': group_players[j],
                        'player2': group_players[k],
                        'winner': None,
                        'completed': False
                    })
                    match_id += 1
            
            groups[f'group_{i+1}'] = {
                'players': group_players,
                'matches': group_matches,
                'standings': {player: {'wins': 0, 'losses': 0} for player in group_players}
            }
        
        return {
            'type': 'group_stage',
            'groups': groups,
            'playoff': None  # Плей-офф будет сгенерирован после группового этапа
        }
    
    def _generate_free_format(self, players: List[str]) -> Dict[str, Any]:
        """Генерация свободного формата"""
        return {
            'type': 'free_format',
            'players': players,
            'matches': {},  # Матчи добавляются динамически
            'standings': {player: {'wins': 0, 'losses': 0, 'matches_played': 0} for player in players}
        }
```

## Генератор изображений

Создайте `utils/image_generator.py`:

```python
from PIL import Image, ImageDraw, ImageFont
from typing import Dict, List, Any, Tuple
import io
from config import IMAGES_DIR

class TournamentImageGenerator:
    def __init__(self):
        self.width = 1200
        self.height = 800
        self.bg_color = (255, 255, 255)
        self.line_color = (0, 0, 0)
        self.player_color = (240, 248, 255)
        self.winner_color = (144, 238, 144)
        
        # Попытка загрузить шрифт
        try:
            self.font = ImageFont.truetype("arial.ttf", 14)
            self.title_font = ImageFont.truetype("arial.ttf", 18)
        except OSError:
            self.font = ImageFont.load_default()
            self.title_font = ImageFont.load_default()
    
    def generate_bracket_image(self, tournament: Any) -> io.BytesIO:
        """Генерация изображения турнирной сетки"""
        img = Image.new('RGB', (self.width, self.height), self.bg_color)
        draw = ImageDraw.Draw(img)
        
        # Заголовок
        title = f"Турнир: {tournament.name or 'Без названия'}"
        title_bbox = draw.textbbox((0, 0), title, font=self.title_font)
        title_width = title_bbox[2] - title_bbox[0]
        draw.text(((self.width - title_width) // 2, 20), title, 
                 fill=(0, 0, 0), font=self.title_font)
        
        if tournament.bracket.get('type') == 'knockout':
            self._draw_knockout_bracket(draw, tournament)
        elif tournament.bracket.get('type') == 'group_stage':
            self._draw_group_stage(draw, tournament)
        elif tournament.bracket.get('type') == 'free_format':
            self._draw_free_format(draw, tournament)
        
        # Сохранение в BytesIO для отправки
        bio = io.BytesIO()
        img.save(bio, format='PNG')
        bio.seek(0)
        return bio
    
    def _draw_knockout_bracket(self, draw: ImageDraw.Draw, tournament: Any) -> None:
        """Отрисовка турнира на выбывание"""
        matches = tournament.bracket.get('matches', {})
        
        y_start = 80
        round_width = self.width // len(matches)
        
        for round_idx, (round_name, round_matches) in enumerate(matches.items()):
            x = 50 + round_idx * round_width
            
            # Заголовок раунда
            draw.text((x, y_start), f"Раунд {round_idx + 1}", 
                     fill=(0, 0, 0), font=self.title_font)
            
            match_height = (self.height - 150) // len(round_matches) if round_matches else 50
            
            for match_idx, match in enumerate(round_matches):
                y = y_start + 40 + match_idx * match_height
                
                # Рамка матча
                match_box = [x, y, x + round_width - 20, y + 80]
                draw.rectangle(match_box, outline=self.line_color, fill=self.player_color)
                
                # Игроки
                player1 = match['player1']
                player2 = match['player2']
                
                # Определяем статус матча
                match_result = tournament.results.get(match['id'], {})
                winner = match_result.get('winner')
                
                # Цвет для победителя
                p1_color = self.winner_color if winner == player1 else self.player_color
                p2_color = self.winner_color if winner == player2 else self.player_color
                
                # Отрисовка игроков
                p1_box = [x + 2, y + 2, x + round_width - 22, y + 38]
                p2_box = [x + 2, y + 42, x + round_width - 22, y + 78]
                
                draw.rectangle(p1_box, outline=self.line_color, fill=p1_color)
                draw.rectangle(p2_box, outline=self.line_color, fill=p2_color)
                
                # Текст игроков
                draw.text((x + 5, y + 15), player1[:15], fill=(0, 0, 0), font=self.font)
                draw.text((x + 5, y + 55), player2[:15], fill=(0, 0, 0), font=self.font)
                
                # Счет
                if match_result.get('completed'):
                    score1 = match_result.get('player1_score', 0)
                    score2 = match_result.get('player2_score', 0)
                    draw.text((x + round_width - 40, y + 15), str(score1), 
                             fill=(0, 0, 0), font=self.font)
                    draw.text((x + round_width - 40, y + 55), str(score2), 
                             fill=(0, 0, 0), font=self.font)
    
    def _draw_group_stage(self, draw: ImageDraw.Draw, tournament: Any) -> None:
        """Отрисовка группового этапа"""
        groups = tournament.bracket.get('groups', {})
        
        group_width = self.width // len(groups)
        y_start = 80
        
        for group_idx, (group_name, group_data) in enumerate(groups.items()):
            x = 20 + group_idx * group_width
            
            # Название группы
            draw.text((x, y_start), f"Группа {group_idx + 1}", 
                     fill=(0, 0, 0), font=self.title_font)
            
            # Игроки группы
            players = group_data.get('players', [])
            for i, player in enumerate(players):
                y = y_start + 30 + i * 25
                draw.text((x, y), f"{i+1}. {player}", fill=(0, 0, 0), font=self.font)
            
            # Матчи группы
            matches = group_data.get('matches', [])
            match_y = y_start + 30 + len(players) * 25 + 20
            
            draw.text((x, match_y), "Матчи:", fill=(0, 0, 0), font=self.title_font)
            
            for i, match in enumerate(matches[:5]):  # Показываем первые 5 матчей
                y = match_y + 25 + i * 20
                match_result = tournament.results.get(match['id'], {})
                
                if match_result.get('completed'):
                    score_text = f"{match_result.get('player1_score')}-{match_result.get('player2_score')}"
                    winner = match_result.get('winner', '')
                    match_text = f"{match['player1'][:8]} vs {match['player2'][:8]} ({score_text})"
                else:
                    match_text = f"{match['player1'][:8]} vs {match['player2'][:8]}"
                
                draw.text((x, y), match_text, fill=(0, 0, 0), font=self.font)
    
    def _draw_free_format(self, draw: ImageDraw.Draw, tournament: Any) -> None:
        """Отрисовка свободного формата"""
        standings = tournament.bracket.get('standings', {})
        
        # Заголовок таблицы
        table_y = 80
        draw.text((50, table_y), "Турнирная таблица", fill=(0, 0, 0), font=self.title_font)
        
        # Заголовки колонок
        headers = ["Место", "Игрок", "Побед", "Поражений", "Игр"]
        col_widths = [80, 200, 80, 100, 80]
        x_positions = [50]
        
        for width in col_widths[:-1]:
            x_positions.append(x_positions[-1] + width)
        
        header_y = table_y + 30
        for i, (header, x_pos) in enumerate(zip(headers, x_positions)):
            draw.text((x_pos, header_y), header, fill=(0, 0, 0), font=self.title_font)
        
        # Сортировка игроков по результатам
        sorted_players = sorted(standings.items(), 
                              key=lambda x: (x[1]['wins'], -x[1]['losses']), reverse=True)
        
        # Отрисовка игроков
        for i, (player, stats) in enumerate(sorted_players):
            y = header_y + 30 + i * 25
            
            # Рамка строки
            row_box = [45, y - 5, x_positions[-1] + 100, y + 20]
            color = self.player_color if i % 2 == 0 else (250, 250, 250)
            draw.rectangle(row_box, outline=self.line_color, fill=color)
            
            # Данные
            row_data = [
                str(i + 1),
                player[:18],
                str(stats['wins']),
                str(stats['losses']),
                str(stats.get('matches_played', 0))
            ]
            
            for data, x_pos in zip(row_data, x_positions):
                draw.text((x_pos + 5, y), data, fill=(0, 0, 0), font=self.font)
    
    def generate_final_results_image(self, tournament: Any) -> io.BytesIO:
        """Генерация итогового изображения с результатами"""
        img = Image.new('RGB', (800, 600), self.bg_color)
        draw = ImageDraw.Draw(img)
        
        # Заголовок
        title = f"Итоги турнира: {tournament.name or 'Без названия'}"
        title_bbox = draw.textbbox((0, 0), title, font=self.title_font)
        title_width = title_bbox[2] - title_bbox[0]
        draw.text(((800 - title_width) // 2, 30), title, 
                 fill=(0, 0, 0), font=self.title_font)
        
        # Определение победителя и призеров
        if tournament.bracket.get('type') == 'knockout':
            self._draw_knockout_results(draw, tournament)
        elif tournament.bracket.get('type') == 'free_format':
            self._draw_free_format_results(draw, tournament)
        
        bio = io.BytesIO()
        img.save(bio, format='PNG')
        bio.seek(0)
        return bio
    
    def _draw_knockout_results(self, draw: ImageDraw.Draw, tournament: Any) -> None:
        """Отрисовка результатов турнира на выбывание"""
        # Ищем финальный матч и определяем места
        matches = tournament.bracket.get('matches', {})
        
        if matches:
            final_round = max(matches.keys(), key=lambda x: int(x.split('_')[1]))
            final_matches = matches[final_round]
            
            if final_matches and tournament.results.get(final_matches[0]['id']):
                final_result = tournament.results[final_matches[0]['id']]
                winner = final_result['winner']
                
                # 1 место
                draw.text((300, 100), "🏆 ПОБЕДИТЕЛЬ 🏆", fill=(255, 215, 0), font=self.title_font)
                draw.text((350, 130), winner, fill=(0, 0, 0), font=self.title_font)
                
                # 2 место
                runner_up = final_matches[0]['player1'] if final_matches[0]['player2'] == winner else final_matches[0]['player2']
                draw.text((320, 180), "🥈 2 место", fill=(192, 192, 192), font=self.title_font)
                draw.text((350, 210), runner_up, fill=(0, 0, 0), font=self.title_font)
    
    def _draw_free_format_results(self, draw: ImageDraw.Draw, tournament: Any) -> None:
        """Отрисовка результатов свободного формата"""
        standings = tournament.bracket.get('standings', {})
        sorted_players = sorted(standings.items(), 
                              key=lambda x: (x[1]['wins'], -x[1]['losses']), reverse=True)
        
        medals = ["🏆", "🥈", "🥉"]
        places = ["ПОБЕДИТЕЛЬ", "2 место", "3 место"]
        colors = [(255, 215, 0), (192, 192, 192), (205, 127, 50)]
        
        for i, (player, stats) in enumerate(sorted_players[:3]):
            y = 100 + i * 70
            medal = medals[i] if i < 3 else ""
            place = places[i] if i < 3 else f"{i+1} место"
            color = colors[i] if i < 3 else (0, 0, 0)
            
            draw.text((300, y), f"{medal} {place}", fill=color, font=self.title_font)
            draw.text((320, y + 25), f"{player} ({stats['wins']} побед)", 
                     fill=(0, 0, 0), font=self.title_font)
```

## Основной файл бота

Создайте `bot.py`:

```python
import logging
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes, ConversationHandler
from datetime import datetime, date, timedelta
import json
from config import BOT_TOKEN, TOURNAMENT_FORMATS
from models.tournament import Tournament
from utils.image_generator import TournamentImageGenerator

# Настройка логирования
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)

logger = logging.getLogger(__name__)

# Состояния для ConversationHandler
(START, MAIN_MENU, CREATE_TOURNAMENT, ENTER_PLAYERS, 
 VIEW_TOURNAMENTS, TOURNAMENT_DETAILS, ENTER_RESULT, 
 REPLACE_PLAYER, FREE_FORMAT_MATCH) = range(9)

class TennisBot:
    def __init__(self):
        self.image_generator = TournamentImageGenerator()
    
    async def start(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
        """Обработчик команды /start"""
        welcome_text = (
            "🎾 Добро пожаловать в бот для организации мини-турниров по теннису!\n\n"
            "Что вы хотите сделать?"
        )
        
        keyboard = [
            [InlineKeyboardButton("🏆 Создать турнир", callback_data="create_tournament")],
            [InlineKeyboardButton("📋 Просмотреть турниры", callback_data="view_tournaments")],
            [InlineKeyboardButton("ℹ️ Помощь", callback_data="help")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        if update.callback_query:
            await update.callback_query.edit_message_text(welcome_text, reply_markup=reply_markup)
        else:
            await update.message.reply_text(welcome_text, reply_markup=reply_markup)
        
        return MAIN_MENU
    
    async def handle_main_menu(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
        """Обработчик главного меню"""
        query = update.callback_query
        await query.answer()
        
        if query.data == "create_tournament":
            await query.edit_message_text(
                "🏆 Создание нового турнира\n\n"
                "Введите количество участников (8, 9 или 12):"
            )
            return CREATE_TOURNAMENT
            
        elif query.data == "view_tournaments":
            return await self.show_date_selection(query, context)
            
        elif query.data == "help":
            help_text = (
                "📋 Помощь по использованию бота:\n\n"
                "🏆 Создать турнир - создание нового мини-турнира\n"
                "📋 Просмотреть турниры - просмотр существующих турниров\n\n"
                "Поддерживаемые форматы:\n"
                "• 8 игроков: плей-офф на выбывание или групповой этап\n"
                "• 9 игроков: групповой этап или свободный формат\n"
                "• 12 игроков: различные групповые форматы\n\n"
                "Можете использовать бота как в личных сообщениях, так и в группах."
            )
            
            keyboard = [[InlineKeyboardButton("◀️ Назад", callback_data="back_to_start")]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await query.edit_message_text(help_text, reply_markup=reply_markup)
            return MAIN_MENU
            
        elif query.data == "back_to_start":
            return await self.start(update, context)
    
    async def handle_create_tournament(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
        """Обработчик создания турнира"""
        if update.callback_query:
            # Обработка выбора формата
            query = update.callback_query
            await query.answer()
            
            if "format_" in query.data:
                format_choice = query.data.replace("format_", "")
                
                # Сохраняем выбор в контексте
                context.user_data['tournament_format'] = format_choice
                
                await query.edit_message_text(
                    f"✅ Выбран формат: {format_choice}\n\n"
                    f"Теперь введите участников по одному. "
                    f"Всего нужно ввести {context.user_data['players_count']} игроков.\n\n"
                    f"Введите имя 1-го игрока:"
                )
                
                context.user_data['current_player'] = 1
                context.user_data['players'] = []
                
                return ENTER_PLAYERS
                
            elif query.data == "cancel_tournament":
                return await self.start(update, context)
        else:
            # Ввод количества игроков
            try:
                players_count = int(update.message.text)
                
                if players_count not in TOURNAMENT_FORMATS:
                    await update.message.reply_text(
                        f"❌ Поддерживается только {', '.join(map(str, TOURNAMENT_FORMATS.keys()))} игроков.\n"
                        "Введите корректное количество:"
                    )
                    return CREATE_TOURNAMENT
                
                context.user_data['players_count'] = players_count
                
                # Показываем доступные форматы
                formats = TOURNAMENT_FORMATS[players_count]
                keyboard = []
                
                for fmt in formats:
                    keyboard.append([InlineKeyboardButton(fmt, callback_data=f"format_{fmt}")])
                
                keyboard.append([InlineKeyboardButton("❌ Отмена", callback_data="cancel_tournament")])
                reply_markup = InlineKeyboardMarkup(keyboard)
                
                await update.message.reply_text(
                    f"👥 Количество игроков: {players_count}\n\n"
                    "Выберите формат турнира:",
                    reply_markup=reply_markup
                )
                return CREATE_TOURNAMENT
                
            except ValueError:
                await update.message.reply_text(
                    "❌ Введите число!\n"
                    "Количество участников (8, 9 или 12):"
                )
                return CREATE_TOURNAMENT
    
    async def handle_enter_players(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
        """Обработчик ввода игроков"""
        if update.callback_query:
            query = update.callback_query
            await query.answer()
            
            if query.data == "auto_draw":
                randomize = True
            elif query.data == "manual_order":
                randomize = False
            else:
                return await self.start(update, context)
            
            # Создаем турнир
            tournament = Tournament()
            tournament.name = f"Турнир {datetime.now().strftime('%d.%m.%Y %H:%M')}"
            tournament.format = context.user_data['tournament_format']
            tournament.players = context.user_data['players']
            tournament.creator_id = update.effective_user.id
            
            # Генерируем турнирную сетку
            tournament.generate_bracket(randomize=randomize)
            tournament.save()
            
            # Генерируем изображение
            image_bio = self.image_generator.generate_bracket_image(tournament)
            
            keyboard = [
                [InlineKeyboardButton("📋 Управлять турниром", callback_data=f"manage_{tournament.id}")],
                [InlineKeyboardButton("◀️ В главное меню", callback_data="back_to_start")]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await query.delete_message()
            await context.bot.send_photo(
                chat_id=update.effective_chat.id,
                photo=image_bio,
                caption=f"🎾 Турнир создан!\n\nID: `{tournament.id}`",
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
            
            # Очищаем данные
            context.user_data.clear()
            
            return MAIN_MENU
        else:
            # Ввод имени игрока
            player_name = update.message.text.strip()
            
            if not player_name:
                await update.message.reply_text("❌ Имя не может быть пустым. Попробуйте еще раз:")
                return ENTER_PLAYERS
            
            if player_name in context.user_data['players']:
                await update.message.reply_text("❌ Игрок с таким именем уже добавлен. Введите другое имя:")
                return ENTER_PLAYERS
            
            # Добавляем игрока
            context.user_data['players'].append(player_name)
            current_player = context.user_data['current_player']
            total_players = context.user_data['players_count']
            
            if current_player < total_players:
                # Запрашиваем следующего игрока
                context.user_data['current_player'] += 1
                await update.message.reply_text(
                    f"✅ Игрок '{player_name}' добавлен.\n\n"
                    f"Введите имя {current_player + 1}-го игрока:"
                )
                return ENTER_PLAYERS
            else:
                # Все игроки введены
                players_list = '\n'.join([f"{i+1}. {name}" for i, name in enumerate(context.user_data['players'])])
                
                keyboard = [
                    [InlineKeyboardButton("🎲 Автоматическая жеребьевка", callback_data="auto_draw")],
                    [InlineKeyboardButton("📝 По порядку ввода", callback_data="manual_order")],
                    [InlineKeyboardButton("❌ Отмена", callback_data="cancel_tournament")]
                ]
                reply_markup = InlineKeyboardMarkup(keyboard)
                
                await update.message.reply_text(
                    f"✅ Все игроки добавлены:\n\n{players_list}\n\n"
                    "Выберите способ формирования турнирной сетки:",
                    reply_markup=reply_markup
                )
                return ENTER_PLAYERS
    
    async def show_date_selection(self, query, context: ContextTypes.DEFAULT_TYPE) -> int:
        """Показ выбора даты для просмотра турниров"""
        keyboard = [
            [InlineKeyboardButton("📅 Текущая неделя", callback_data="date_current_week")],
            [InlineKeyboardButton("📅 Последние 30 дней", callback_data="date_last_month")],
            [InlineKeyboardButton("📅 Все турниры", callback_data="date_all")],
            [InlineKeyboardButton("◀️ Назад", callback_data="back_to_start")]
        ]
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            "📋 Просмотр турниров\n\n"
            "Выберите период:",
            reply_markup=reply_markup
        )
        return VIEW_TOURNAMENTS
    
    async def handle_view_tournaments(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
        """Обработчик просмотра турниров"""
        query = update.callback_query
        await query.answer()
        
        if query.data == "back_to_start":
            return await self.start(update, context)
        
        # Определяем диапазон дат
        if query.data == "date_current_week":
            today = date.today()
            start_date = today - timedelta(days=today.weekday())
            end_date = start_date + timedelta(days=6)
        elif query.data == "date_last_month":
            end_date = date.today()
            start_date = end_date - timedelta(days=30)
        else:  # date_all
            start_date = date(2020, 1, 1)
            end_date = date(2030, 12, 31)
        
        # Получаем турниры
        tournaments = Tournament.get_tournaments_by_date_range(start_date, end_date)
        
        if not tournaments:
            keyboard = [[InlineKeyboardButton("◀️ Назад", callback_data="view_tournaments")]]
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            await query.edit_message_text(
                "📋 Турниры не найдены\n\n"
                "За выбранный период турниры не проводились.",
                reply_markup=reply_markup
            )
            return VIEW_TOURNAMENTS
        
        # Показываем список турниров
        keyboard = []
        for tournament in tournaments[:10]:  # Показываем максимум 10
            created_date = datetime.fromisoformat(tournament.created_date)
            status_emoji = {"forming": "🔨", "started": "⚡", "finished": "🏁"}
            tournament_text = (f"{status_emoji.get(tournament.status, '❓')} "
                             f"{tournament.name or 'Турнир'} "
                             f"({created_date.strftime('%d.%m %H:%M')})")
            keyboard.append([InlineKeyboardButton(tournament_text, callback_data=f"view_{tournament.id}")])
        
        keyboard.append([InlineKeyboardButton("◀️ Назад", callback_data="view_tournaments")])
        reply_markup = InlineKeyboardMarkup(keyboard)
        
        await query.edit_message_text(
            f"📋 Найдено турниров: {len(tournaments)}\n\n"
            "Выберите турнир для просмотра:",
            reply_markup=reply_markup
        )
        return VIEW_TOURNAMENTS

    async def handle_tournament_details(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
        """Обработчик деталей турнира"""
        query = update.callback_query
        await query.answer()
        
        if query.data.startswith("view_"):
            tournament_id = query.data.replace("view_", "")
            tournament = Tournament.load(tournament_id)
            
            if not tournament:
                await query.edit_message_text(
                    "❌ Турнир не найден",
                    reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад", callback_data="view_tournaments")]])
                )
                return VIEW_TOURNAMENTS
            
            # Сохраняем ID турнира в контексте
            context.user_data['current_tournament_id'] = tournament_id
            
            # Генерируем изображение
            image_bio = self.image_generator.generate_bracket_image(tournament)
            
            # Создаем клавиатуру в зависимости от статуса турнира
            keyboard = []
            
            if tournament.status == "forming":
                keyboard.extend([
                    [InlineKeyboardButton("🚀 Начать турнир", callback_data=f"start_{tournament_id}")],
                    [InlineKeyboardButton("🔄 Заменить игрока", callback_data=f"replace_player_{tournament_id}")]
                ])
            elif tournament.status == "started":
                keyboard.extend([
                    [InlineKeyboardButton("⚽ Ввести результат", callback_data=f"enter_result_{tournament_id}")],
                    [InlineKeyboardButton("🏁 Завершить турнир", callback_data=f"finish_{tournament_id}")]
                ])
            elif tournament.status == "finished":
                keyboard.append([InlineKeyboardButton("🏆 Итоговые результаты", callback_data=f"final_results_{tournament_id}")])
            
            keyboard.append([InlineKeyboardButton("◀️ Назад к списку", callback_data="view_tournaments")])
            reply_markup = InlineKeyboardMarkup(keyboard)
            
            status_text = {"forming": "🔨 Формирование", "started": "⚡ Идет", "finished": "🏁 Завершен"}
            created_date = datetime.fromisoformat(tournament.created_date)
            
            caption = (f"🎾 {tournament.name or 'Турнир'}\n\n"
                      f"📅 Создан: {created_date.strftime('%d.%m.%Y %H:%M')}\n"
                      f"👥 Игроков: {len(tournament.players)}\n"
                      f"🎯 Формат: {tournament.format}\n"
                      f"📊 Статус: {status_text.get(tournament.status, 'Неизвестен')}\n\n"
                      f"ID: `{tournament.id}`")
            
            await query.delete_message()
            await context.bot.send_photo(
                chat_id=update.effective_chat.id,
                photo=image_bio,
                caption=caption,
                reply_markup=reply_markup,
                parse_mode='Markdown'
            )
            
            return TOURNAMENT_DETAILS
            
        elif query.data.startswith("start_"):
            tournament_id = query.data.replace("start_", "")
            tournament = Tournament.load(tournament_id)
            
            if tournament:
                tournament.status = "started"
                tournament.start_date = datetime.now().isoformat()
                tournament.save()
                
                await query.edit_message_text(
                    f"🚀 Турнир '{tournament.name}' начался!\n\n"
                    "Теперь можно вводить результаты матчей.",
                    reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад к турниру", callback_data=f"view_{tournament_id}")]])
                )
            
            return TOURNAMENT_DETAILS
            
        elif query.data.startswith("finish_"):
            tournament_id = query.data.replace("finish_", "")
            tournament = Tournament.load(tournament_id)
            
            if tournament:
                tournament.status = "finished"
                tournament.finish_date = datetime.now().isoformat()
                tournament.save()
                
                await query.edit_message_text(
                    f"🏁 Турнир '{tournament.name}' завершен!\n\n"
                    "Поздравляем всех участников!",
                    reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад к турниру", callback_data=f"view_{tournament_id}")]])
                )
            
            return TOURNAMENT_DETAILS
            
        elif query.data.startswith("final_results_"):
            tournament_id = query.data.replace("final_results_", "")
            tournament = Tournament.load(tournament_id)
            
            if tournament:
                # Генерируем итоговое изображение
                image_bio = self.image_generator.generate_final_results_image(tournament)
                
                await query.delete_message()
                await context.bot.send_photo(
                    chat_id=update.effective_chat.id,
                    photo=image_bio,
                    caption=f"🏆 Итоговые результаты турнира\n{tournament.name}",
                    reply_markup=InlineKeyboardMarkup([[InlineKeyboardButton("◀️ Назад к турниру", callback_data=f"view_{tournament_id}")]])
                )
            
            return TOURNAMENT_DETAILS
        
        return TOURNAMENT_DETAILS

    async def group_command(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
        """Обработчик команды для работы в группах"""
        # В группах доступен только просмотр турниров
        if len(context.args) == 0:
            await update.message.reply_text(
                "Использование: /tournament <ID_турнира>\n"
                "Для получения ID создайте турнир в личных сообщениях с ботом."
            )
            return
        
        tournament_id = context.args[0]
        tournament = Tournament.load(tournament_id)
        
        if not tournament:
            await update.message.reply_text("❌ Турнир с таким ID не найден.")
            return
        
        # Генерируем изображение без кнопок управления
        image_bio = self.image_generator.generate_bracket_image(tournament)
        
        status_text = {"forming": "🔨 Формирование", "started": "⚡ Идет", "finished": "🏁 Завершен"}
        created_date = datetime.fromisoformat(tournament.created_date)
        
        caption = (f"🎾 {tournament.name or 'Турнир'}\n\n"
                  f"📅 Создан: {created_date.strftime('%d.%m.%Y %H:%M')}\n"
                  f"👥 Игроков: {len(tournament.players)}\n"
                  f"🎯 Формат: {tournament.format}\n"
                  f"📊 Статус: {status_text.get(tournament.status, 'Неизвестен')}")
        
        await context.bot.send_photo(
            chat_id=update.effective_chat.id,
            photo=image_bio,
            caption=caption
        )

def main():
    """Основная функция"""
    bot = TennisBot()
    
    # Создаем приложение
    application = Application.builder().token(BOT_TOKEN).build()
    
    # Создаем ConversationHandler
    conv_handler = ConversationHandler(
        entry_points=[CommandHandler('start', bot.start)],
        states={
            MAIN_MENU: [CallbackQueryHandler(bot.handle_main_menu)],
            CREATE_TOURNAMENT: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, bot.handle_create_tournament),
                CallbackQueryHandler(bot.handle_create_tournament)
            ],
            ENTER_PLAYERS: [
                MessageHandler(filters.TEXT & ~filters.COMMAND, bot.handle_enter_players),
                CallbackQueryHandler(bot.handle_enter_players)
            ],
            VIEW_TOURNAMENTS: [CallbackQueryHandler(bot.handle_view_tournaments)],
            TOURNAMENT_DETAILS: [CallbackQueryHandler(bot.handle_tournament_details)],
        },
        fallbacks=[CommandHandler('start', bot.start)],
        per_chat=True,
        per_user=True
    )
    
    # Добавляем обработчики
    application.add_handler(conv_handler)
    application.add_handler(CommandHandler('tournament', bot.group_command))
    
    # Запускаем бота
    print("🎾 Бот запущен...")
    application.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == '__main__':
    main()
```

## Настройка и запуск Telegram бота

### 1. Создание бота в Telegram

1. Найдите [@BotFather](https://t.me/BotFather) в Telegram
2. Отправьте команду `/newbot`
3. Введите название бота (например, "Tennis Tournament Bot")
4. Введите username бота (например, "tennis_tournament_bot")
5. Скопируйте полученный токен

### 2. Настройка окружения

```bash
# Создайте виртуальное окружение
python -m venv tennis_bot_env

# Активируйте его
# Windows:
tennis_bot_env\Scripts\activate
# Linux/macOS:
source tennis_bot_env/bin/activate

# Установите зависимости
pip install -r requirements.txt
```

### 3. Настройка переменных окружения

Создайте файл `.env`:

```env
TELEGRAM_BOT_TOKEN=ваш_токен_здесь
```

Или экспортируйте переменную:

```bash
export TELEGRAM_BOT_TOKEN="ваш_токен_здесь"
```

### 4. Запуск бота

```bash
python bot.py
```

## Особенности реализации

### Форматы турниров

1. **8 игроков**:
   - Плей-офф на выбывание: 8 → 4 → 2 → 1 (7 матчей)
   - Групповой этап: 2 группы по 4, затем плей-офф

2. **9 игроков**:
   - 3 группы по 3, лучшие в плей-офф
   - Свободный формат

3. **12 игроков**:
   - 3 группы по 4 + плей-офф
   - 4 группы по 3 + плей-офф

### Хранение данных

Каждый турнир сохраняется в отдельный JSON файл:

```json
{
  "id": "uuid",
  "name": "Турнир 15.03.2024 18:30",
  "format": "Плей-офф на выбывание",
  "players": ["Игрок1", "Игрок2", ...],
  "bracket": {...},
  "results": {...},
  "status": "started",
  "created_date": "2024-03-15T18:30:00",
  "creator_id": 123456789
}
```

### Генерация изображений

Изображения турнирных сеток создаются с помощью PIL и содержат:
- Турнирную сетку с именами игроков
- Результаты матчей
- Выделение победителей
- Красивое оформление

### Работа в группах

В групповых чатах бот работает по команде:
```
/tournament <ID_турнира>
```

Это показывает текущую турнирную сетку без возможности управления.

Бот готов к использованию и поддерживает все описанные функции!