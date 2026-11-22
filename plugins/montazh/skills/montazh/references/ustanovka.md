# Инструменты для монтажа: проверка и установка

Что нужно и зачем:

| Что | Зачем | Mac | Windows |
|---|---|---|---|
| ffmpeg (с субтитрами, фильтр `ass`) | режет, склеивает, рисует субтитры, считает громкость | `brew install ffmpeg-full`, затем `brew unlink ffmpeg` и `brew link --force ffmpeg-full` (обычная формула `ffmpeg` бывает без libass) | `winget install --id Gyan.FFmpeg -e` (полная сборка с субтитрами) |
| Node 22.20+ | ставит навыки (HyperFrames, Remotion, GSAP) | `brew install node` | `winget install --id OpenJS.NodeJS.LTS -e` |
| uv | запускает скрипты плагина и расшифровку (сам скачивает Python) | `brew install uv` | `winget install --id astral-sh.uv -e` |
| Навыки HyperFrames (по желанию, режим «Про») | анимированные субтитры и моушн-вставки | `npx skills add heygen-com/hyperframes -g -a claude-code` (только по «да») | то же |

## Проверка (только читает)

Mac:

```bash
export PATH="$HOME/.local/bin:/opt/homebrew/bin:/usr/local/bin:$PATH"
has_skill() { for d in "$HOME/.claude/skills/$1" "$HOME/.claude/skills/synced"/*/"$1"; do [ -f "$d/SKILL.md" ] && return 0; done; return 1; }
show() { name="$1"; shift; if "$@" >/dev/null 2>&1; then echo "$name: стоит"; else echo "$name: НЕ СТОИТ"; fi; }
show ffmpeg command -v ffmpeg
if command -v ffmpeg >/dev/null; then ffmpeg -hide_banner -filters 2>/dev/null | grep -qE ' (ass|subtitles) ' && echo "ffmpeg, субтитры: стоит" || echo "ffmpeg, субтитры: НЕТ (нужна сборка с libass: ffmpeg-full)"; fi
show ffmpeg-skill has_skill ffmpeg-skill
if command -v node >/dev/null && node -e 'const [a,b]=process.versions.node.split(".").map(Number);process.exit(a>22||(a===22&&b>=20)?0:1)'; then echo "node: стоит ($(node -v))"; else echo "node: НЕ СТОИТ или старше 22.20"; fi
show uv command -v uv
show hyperframes has_skill hyperframes
show remotion has_skill remotion-best-practices
show gsap has_skill gsap-core
```

Windows (PowerShell):

```powershell
function Show($n, $ok) { if ($ok) { "${n}: стоит" } else { "${n}: НЕ СТОИТ" } }
function Has-Skill($n) { (Test-Path -LiteralPath "$HOME\.claude\skills\$n\SKILL.md") -or (@(Get-ChildItem -Path "$HOME\.claude\skills\synced\*\$n\SKILL.md" -ErrorAction SilentlyContinue).Count -gt 0) }
Show 'ffmpeg' [bool](Get-Command ffmpeg -ErrorAction SilentlyContinue)
Show 'node' [bool](Get-Command node -ErrorAction SilentlyContinue)
Show 'uv' [bool](Get-Command uv -ErrorAction SilentlyContinue)
Show 'hyperframes' (Has-Skill 'hyperframes')
Show 'remotion' (Has-Skill 'remotion-best-practices')
Show 'gsap' (Has-Skill 'gsap-core')
```

Проверка «есть ли у ffmpeg субтитры» на Windows: `ffmpeg -hide_banner -filters | Select-String ' (ass|subtitles) '` (сборка Gyan.FFmpeg полная, libass в ней есть; если строки нет, значит в PATH стоит другой ffmpeg: `where.exe ffmpeg`).

## Что делать, если чего-то нет

- **ffmpeg, Node, uv** нужны для монтажа: ставишь сам командами из таблицы, потом проверяешь `--version` и одной строкой говоришь, что поставил. Mac без Homebrew: не запускай скрипты установки из интернета, скажи: «Поставь Homebrew с brew.sh одной командой с сайта, потом я допоставлю остальное». Windows: после `winget` закрой и открой приложение, чтобы подхватился PATH. Если `ffmpeg` стоит, но без фильтров `ass`/`subtitles`: на Mac `brew install ffmpeg-full`, затем `brew unlink ffmpeg` и `brew link --force ffmpeg-full` (вернуть: `brew unlink ffmpeg-full && brew link ffmpeg`); скрипты сами находят полную сборку в `/opt/homebrew/opt/ffmpeg-full/bin`. Это занимает несколько минут: предупреди.
- **HyperFrames** (режим «Про») — только по «да» пользователя. Без него монтаж идёт в базовом режиме, всё работает.
- Всё остальное вне этого списка ставится только после «да» пользователя.

## Маска человека для стиля «Афиша» (по желанию, только по «да»)

Нужна, чтобы слово-тема стояло ЗА человеком. Это Robust Video Matting (RVM, MobileNetV3, ONNX): вырезает человека по волосам, считается на обычном процессоре (15 секунд видео — 1–4 минуты). Ставятся два куска: окружение с `onnxruntime` (~60 МБ, версии закреплены в `references/mask-requirements.txt`, ставится с `--no-deps`) и модель `rvm_mobilenetv3_fp32.onnx` (15 МБ), которую скачивает `scripts/install_rvm.py` с официального релиза авторов (PeterL1n/RobustVideoMatting, v1.0.0) только по https со сверкой размера и SHA256. **Лицензия RVM: GPL-3.0.** Модель не лежит в плагине: пользователь скачивает её с сайта авторов для себя. Ключей и аккаунтов нет.

Mac и Linux (нужен `uv` и Python 3.12):

```bash
S="$(find ~/.claude/plugins -path '*skills/montazh/scripts' -maxdepth 8 | sort | tail -1)"
R="$S/../references/mask-requirements.txt"
M="$HOME/.montazh/mask"
mkdir -p "$HOME/.montazh"
T="$(mktemp -d)"; cd "$T"
export UV_NO_CONFIG=1
uv venv "$M" --python 3.12 --no-python-downloads
uv pip install --python "$M/bin/python" --index-url https://pypi.org/simple --no-deps --only-binary :all: -r "$R"
"$M/bin/python" -c "import onnxruntime, numpy; onnxruntime.InferenceSession; print('окружение маски поставлено')"
"$M/bin/python" "$S/install_rvm.py"
cd ~; rmdir "$T"
```

Windows (PowerShell, нужен `uv`): те же шаги — `uv venv "$HOME\.montazh\mask" --python 3.12`, установка пакетов из `mask-requirements.txt` с `--index-url https://pypi.org/simple --no-deps`, затем `"$HOME\.montazh\mask\Scripts\python.exe" <скрипты>\install_rvm.py`.

Без маски стиль «Афиша» работает в запасном виде (слово-тема над головой).
