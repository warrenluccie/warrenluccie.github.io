# 第03课补充说明：与代码实现保持一致

- **全局配置文件**：只使用一个 `config.toml`。
  - 用户级：`~/.config/cherryquant/config.toml`（优先级最高；建议放 token/key 等敏感信息）
  - 项目级：`config/config.toml`（仓库默认配置；建议提交作为示范）
  - 都不存在：使用代码默认值（`CherryQuantSettings()`）
- **系统配置入口**：`src/cherryquant/config/settings.py`，对外提供：
  - `get_settings()`：加载并缓存配置
  - `get_settings_source()`：返回生效配置文件路径
  - `reload_settings()`：清缓存重载
- **更严格的校验**：`src/cherryquant/config/models.py` 使用 Pydantic v2，并开启 `extra="forbid"` + 冻结模型（frozen），写错字段会直接报错。
- **业务配置**：`config/futures_config.toml` 由 `src/cherryquant/config/futures_loader.py` 加载（Pydantic 模型 + 单例缓存）。
- **Prompt 管理**：Prompt 目录优先级：`~/.config/cherryquant/prompts` > `config/prompts`；如需目录 Prompt，可使用 `src/cherryquant/config/prompt_loader.py`。
- **测试建议**：执行 `uv run pytest`（特别是 `tests/test_config/test_settings_merge.py`）验证配置加载优先级与错误提示。
