# Logging in Python — Practical Implementation

*Day X of my learning journal — Krish Naik's Complete Python Bootcamp ([Logging module](https://github.com/krishnaik06/Complete-Python-Bootcamp/tree/main/12-Logging%20In%20Python))*

For the longest time my "debugging strategy" was just sprinkling `print()` everywhere and deleting them later (or forgetting to, and shipping debug prints to production like a menace). This module is basically the professional replacement for that habit. Writing this note to actually lock in *why* logging exists, not just the syntax.

---

## 1. Why not just use `print()`?

`print()` only does one thing — dump text to the console. That's it. It has no concept of:
- **severity** (is this a harmless debug note or a "the app is on fire" error?)
- **destination** (console only — you can't easily also save it to a file)
- **on/off switch** (you can't say "show me only warnings and above" without editing code)
- **timestamps, module names, line numbers** — you'd have to manually type all that every time

`logging` solves all four of these out of the box. That's the whole pitch.

---

## 2. Log levels — the severity ladder

Every log message has a **level**, and they go in this order (low → high severity):

| Level | When to use it |
|-------|-----------------|
| `DEBUG` | detailed info, only useful while actively debugging |
| `INFO` | confirmation that things are working as expected |
| `WARNING` | something unexpected happened, but the app still works |
| `ERROR` | something failed — a function/operation didn't complete |
| `CRITICAL` | serious error — the app itself might not be able to continue |

The important mental model here: when you set a logger's level, you're setting a **minimum threshold**. If I set the level to `WARNING`, then `DEBUG` and `INFO` messages get silently ignored — only `WARNING`, `ERROR`, and `CRITICAL` actually show up. This is what makes logging so much better than `print()` — I can dial the noise up or down with a single line, no need to delete/comment out anything.

---

## 3. The fastest way to start — `basicConfig`

This is the "hello world" of logging. One function call configures everything:

```python
import logging

def basic_logger():
    logging.basicConfig(filename='app.log', level=logging.DEBUG)
    logging.debug('This is a debug message')
    logging.info('This is an info message')
    logging.warning('This is a warning message')
    logging.error('This is an error message')
    logging.critical('This is a critical message')

basic_logger()
```

Because `level=logging.DEBUG` is the lowest rung on the ladder, *everything* above and including `DEBUG` gets written to `app.log`. Change it to `logging.WARNING` and only `warning`, `error`, `critical` show up — `debug` and `info` get dropped.

**Catch worth knowing:** `basicConfig()` only works the *first* time it's called in a process. If a logger's already been configured (even by an imported library), calling it again does nothing. This is exactly why real projects don't rely on `basicConfig` — they build their own logger explicitly, which is the next section.

---

## 4. Building a real logger: Logger → Handler → Formatter

This is the part that actually clicked once I understood these are **three separate, stackable pieces**, not one blob of config:

- **Logger** — the object you actually call `.debug()` / `.info()` etc. on. Think of it as the "reporter."
- **Handler** — decides *where* the log goes (a file? the console? both?). You can attach multiple handlers to one logger.
- **Formatter** — decides *how* the message looks (timestamp, level, message text, etc.) — attached to a handler, not the logger directly.

**Analogy that made this stick for me:** the Logger is a news reporter gathering a story. Handlers are the different channels the story gets published to (newspaper file, live TV/console). The Formatter is the house style each channel prints in — a newspaper article looks different from a TV ticker, even though it's the exact same story.

```python
def logger_with_handlers():
    logger = logging.getLogger('my_logger')
    logger.setLevel(logging.DEBUG)

    file_handler = logging.FileHandler('app.log')
    console_handler = logging.StreamHandler()

    file_handler.setLevel(logging.DEBUG)
    console_handler.setLevel(logging.DEBUG)

    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    file_handler.setFormatter(formatter)
    console_handler.setFormatter(formatter)

    logger.addHandler(file_handler)
    logger.addHandler(console_handler)

    logger.debug('This is a debug message')
    logger.info('This is an info message')
    logger.warning('This is a warning message')
    logger.error('This is an error message')
    logger.critical('This is a critical message')

logger_with_handlers()
```

Notice you can set **different levels per handler**. E.g. console shows everything for live debugging, but the file only saves `WARNING` and above, so the log file doesn't blow up with noise:

```python
file_handler.setLevel(logging.WARNING)     # file: only warnings+
console_handler.setLevel(logging.DEBUG)    # console: everything
```

This is genuinely one of the most useful patterns in this whole module — one logger, two destinations, two different sensitivity levels.

---

## 5. Formatting — making log lines actually readable

A raw log line with no formatter is just the message text, nothing else. A formatter is where you decide what metadata gets attached:

```python
formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
```

Common format placeholders I keep reaching for:

| Placeholder | Meaning |
|---|---|
| `%(asctime)s` | timestamp |
| `%(name)s` | logger's name |
| `%(levelname)s` | DEBUG / INFO / WARNING / etc. |
| `%(message)s` | the actual log text |
| `%(funcName)s` | function the log call happened in |
| `%(lineno)d` | line number of the log call |

And you can absolutely give the file handler and console handler **different formats** — maybe the file gets the full detailed version (for later debugging), and the console gets a short clean version (for readability while the app is running):

```python
file_formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
console_formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')

file_handler.setFormatter(file_formatter)
console_handler.setFormatter(console_formatter)
```

---

## 6. Rotating log files — so your logs don't eat your disk

If an app runs for weeks, a single log file can balloon to gigabytes. `RotatingFileHandler` fixes this by auto-splitting the file once it hits a size limit, and deleting the oldest backups once you have too many:

```python
from logging.handlers import RotatingFileHandler

def logger_with_rotating_file_handler():
    logger = logging.getLogger('rotating_logger')
    logger.setLevel(logging.DEBUG)

    rotating_handler = RotatingFileHandler('rotating_app.log', maxBytes=2000, backupCount=5)
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    rotating_handler.setFormatter(formatter)

    logger.addHandler(rotating_handler)

    for i in range(100):
        logger.debug('This is debug message number {}'.format(i))
```

- `maxBytes=2000` → once the current file hits ~2000 bytes, it rolls over to a new one.
- `backupCount=5` → keeps at most 5 old log files around (`rotating_app.log.1`, `.2`, ... up to `.5`). Once you exceed that, the oldest one gets deleted automatically.

**Analogy:** it's like a gym locker room with only 5 spare lockers — once a new person needs a locker and they're all full, the person who's been in there longest gets kicked out. You never have to manually clean up old logs yourself.

---

## 7. Logging exceptions properly

This is the pattern I'll actually use the most in real projects — instead of an exception silently crashing or getting swallowed, log the **full stack trace**:

```python
def log_exception():
    logger = logging.getLogger('exception_logger')
    logger.setLevel(logging.ERROR)

    file_handler = logging.FileHandler('exception_app.log')
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)

    try:
        1 / 0
    except Exception as e:
        logger.exception("An exception occurred")
```

`logger.exception(...)` is basically `logger.error(...)` with one superpower added: it automatically attaches the full traceback to the log entry. You get to see exactly which line blew up and the call chain that led there, without manually formatting `traceback.format_exc()` yourself. This only makes sense to call **inside an `except` block** — outside of one there's no active exception to attach.

---

## 8. Contextual logging — attaching extra info to every message

Sometimes the message alone isn't enough — you want to know *which user*, *which session*, *which function* triggered a log line. Two levels of this:

**Built-in context** (function name + line number come free via the formatter):

```python
formatter = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s - %(funcName)s - %(lineno)d'
)
```

**Custom context** via the `extra` parameter — this is for data that's specific to *your* app, like a user ID or session ID:

```python
def logger_with_additional_context(user_id, session_id):
    logger = logging.getLogger('additional_context_logger')
    logger.setLevel(logging.DEBUG)

    file_handler = logging.FileHandler('additional_context_app.log')
    formatter = logging.Formatter(
        '%(asctime)s - %(levelname)s - %(message)s - UserID: %(user_id)s - SessionID: %(session_id)s'
    )
    file_handler.setFormatter(formatter)
    logger.addHandler(file_handler)

    extra = {'user_id': user_id, 'session_id': session_id}

    logger.debug('This is a debug message', extra=extra)
    logger.info('This is an info message', extra=extra)

logger_with_additional_context('user123', 'session456')
```

The key rule: any key you put inside `extra={}` **must** also appear in the formatter string as `%(that_key)s`, or Python throws an error. `extra` is essentially injecting custom variables into the format string at log time.

Real use case I can already picture: a Flask/FastAPI app where every log line automatically tags which user and which request triggered it — makes debugging a production issue for "user123" way faster than scrolling through a wall of anonymous log lines.

---

## 9. Multiple loggers — configuring logging with a dictionary

Once an app has more than one file/module, hardcoding handler setup in every file gets messy fast. `logging.config.dictConfig()` lets you define the **entire logging setup as one dictionary** — handlers, formatters, and levels all in one place:

```python
import logging.config

log_config = {
    'version': 1,
    'formatters': {
        'default': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        },
        'detailed': {
            'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s - %(funcName)s - %(lineno)d'
        }
    },
    'handlers': {
        'file': {
            'class': 'logging.FileHandler',
            'filename': 'dict_config_app.log',
            'formatter': 'detailed',
            'level': 'DEBUG'
        },
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'default',
            'level': 'DEBUG'
        }
    },
    'root': {
        'handlers': ['file', 'console'],
        'level': 'DEBUG'
    }
}

logging.config.dictConfig(log_config)
logger = logging.getLogger('')
logger.info('This is an info message')
```

This is basically the same Logger/Handler/Formatter pieces from Section 4, just described declaratively instead of built line-by-line in Python. Way easier to read at a glance, and way easier to swap configs between dev/prod without touching app code.

---

## 10. Real-world example: multi-module app with its own logger per file

This is the pattern that made logging finally feel "real" instead of a toy example — a small app spread across three files, where **every module gets its own named logger**, but they all funnel into one shared config set up once, in `main.py`.

**`main.py`** — sets up logging once, for the whole app:

```python
import logging
import logging.config
from module_a import module_a_function
from module_b import module_b_function

def setup_logging():
    log_config = {
        'version': 1,
        'formatters': {
            'default': {'format': '%(asctime)s - %(name)s - %(levelname)s - %(message)s'}
        },
        'handlers': {
            'file': {
                'class': 'logging.FileHandler',
                'filename': 'multi_module_app.log',
                'formatter': 'default',
                'level': 'DEBUG'
            },
            'console': {
                'class': 'logging.StreamHandler',
                'formatter': 'default',
                'level': 'DEBUG'
            }
        },
        'root': {'handlers': ['file', 'console'], 'level': 'DEBUG'}
    }
    logging.config.dictConfig(log_config)

if __name__ == '__main__':
    setup_logging()
    logger = logging.getLogger(__name__)
    logger.info('Main module started')
    module_a_function()
    module_b_function()
    logger.info('Main module finished')
```

**`module_a.py`** — no setup needed here at all, just grabs a logger with its own name:

```python
import logging

def module_a_function():
    logger = logging.getLogger(__name__)
    logger.info('Module A function started')
    logger.debug('This is a debug message from Module A')
    logger.info('Module A function finished')
```

**`module_b.py`** — same idea:

```python
import logging

def module_b_function():
    logger = logging.getLogger(__name__)
    logger.info('Module B function started')
    logger.debug('This is a debug message from Module B')
    logger.info('Module B function finished')
```

**Why `logging.getLogger(__name__)` is the pattern to always use in real projects:** `__name__` automatically becomes the module's path (e.g. `module_a`, `module_b`). So in your log file, every line already tells you *which file it came from*, for free, with zero extra typing. And because none of the child loggers configure their own handlers, they **propagate** up to the `root` logger set up in `main.py` — one config, applied everywhere. This is exactly how logging is meant to scale: configure once at the entry point of the app, and every module downstream just calls `getLogger(__name__)` and gets it for free.

---

## 11. Advanced: configuring logging from an external file

For bigger projects, even the dictionary config can live outside your Python code entirely, in a `.conf` file — so ops/deployment can tweak logging behavior without touching source code:

**`logging.conf`:**
```ini
[loggers]
keys=root

[handlers]
keys=fileHandler,consoleHandler

[formatters]
keys=defaultFormatter

[logger_root]
level=DEBUG
handlers=fileHandler,consoleHandler

[handler_fileHandler]
class=FileHandler
level=DEBUG
formatter=defaultFormatter
args=('advanced_logging_app.log', 'a')

[handler_consoleHandler]
class=StreamHandler
level=DEBUG
formatter=defaultFormatter
args=(sys.stdout,)

[formatter_defaultFormatter]
format=%(asctime)s - %(name)s - %(levelname)s - %(message)s
```

**Loading it:**
```python
import logging.config

logging.config.fileConfig('logging.conf')
logger = logging.getLogger(__name__)
logger.info('This is an info message')
```

Haven't needed this in a personal project yet, but good to know it exists for when logging config needs to change per-environment without a code deploy.

---

## 12. Performance note

Formatting a log message and writing to disk/console isn't free — it costs a bit of time per call, especially at high volume (I benchmarked 10,000 debug calls across file/console/rotating handlers and formatted vs unformatted messages). Two practical takeaways:

1. **Console handlers are noticeably slower than file handlers** at high volume — printing to a terminal has more overhead than appending to a file. Don't leave a console handler wired up in a hot loop in production.
2. Logging calls below the active threshold aren't fully "free," but they're cheap — the level check happens before the expensive formatting work. Still, don't log `DEBUG` messages inside a tight loop that runs millions of times unless the logger is actually configured at `DEBUG` or lower.

---

## 13. My personal cheat sheet

```python
import logging

logger = logging.getLogger(__name__)   # always use __name__, not a random string
logger.setLevel(logging.DEBUG)         # lowest level this logger will consider

handler = logging.FileHandler('app.log')     # or StreamHandler() for console
handler.setLevel(logging.DEBUG)              # this handler's own threshold

formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
handler.setFormatter(formatter)

logger.addHandler(handler)

logger.debug("details for developers")
logger.info("normal operation checkpoint")
logger.warning("something's off but we're fine")
logger.error("something failed")
logger.critical("app might be dying")

# inside an except block:
logger.exception("something broke, here's the full traceback")
```

Rules going forward:
1. Use `getLogger(__name__)`, never the plain root `logging.debug()` calls, in anything beyond a one-off script.
2. Set up logging **once**, at the app's entry point — every other module just grabs its own named logger.
3. Use `logger.exception()` inside `except` blocks instead of `print(e)` — the stack trace is priceless later.
4. Rotate log files (`RotatingFileHandler`) for anything long-running, so disk space doesn't become a problem.
5. Keep console handlers verbose for dev, but dial file handlers down to `WARNING+` in production so the log file stays useful, not noisy.
