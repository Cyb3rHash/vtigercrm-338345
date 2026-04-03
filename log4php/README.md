# `log4php/` — logging facade/wrappers

vtiger bundles Log4PHP-related helpers here.

Key files:
- `LoggerManager.php`
- `LoggerPropertyConfigurator.php`

Configuration:
- `log4php.properties` at repository root

## Notes for developers
- Many parts of the codebase use `LoggerManager` to obtain loggers.
- Log levels/appenders are configured via `log4php.properties`.
