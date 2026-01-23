## 目标

- 使用 Rime 输入简体中文（如 `luna_pinyin` 方案）
- 同时保留一个原生英文键盘输入法（不经过 Rime，无候选窗）
- 可通过快捷键在两者之间切换

---

## 前提条件

确保已安装必要软件包：

```bash
sudo apt install fcitx5 fcitx5-rime fcitx5-config-qt librime-data rime-tools
```

> 注意：`librime-data` 提供默认输入方案；`rime-tools` 提供 `rime_deployer` 用于部署配置。

---

## 步骤一：配置 Rime 输出简体中文

1. 创建用户配置目录（若不存在）：
    
    ```bash
    mkdir -p ~/.local/share/fcitx5/rime
    ```
    
2. 创建或编辑自定义配置文件：
    
    ```bash
    nano ~/.local/share/fcitx5/rime/default.custom.yaml
    ```
    
3. 写入以下内容（以 `luna_pinyin` 为例）：
    
    ```yaml
    patch:
      schema_list:
        - schema: luna_pinyin
    
      "luna_pinyin.schema/switches/@0/options":
        - zh_simp
    ```
    
4. 部署配置：
    
    ```bash
    rime_deployer --build ~/.local/share/fcitx5/rime
    ```
    

---

## 步骤二：在 Fcitx5 中添加两个输入法

1. 启动配置工具：
    
    ```bash
    fcitx5-configtool
    ```
    
2. 点击左下角 “+” 添加输入法：
    
    - 搜索并添加 **Rime**
    - 搜索并添加 **Keyboard - English (US)**  
        （注意：不要选择任何带 “Rime” 字样的英文方案）
3. 调整顺序（可选），例如：
    
    - 第一项：Rime（简体中文）
    - 第二项：Keyboard - English (US)
4. 关闭配置窗口，配置自动保存。
    

---

## 步骤三：设置输入法切换快捷键（可选）

默认切换快捷键为 `Ctrl + Space`。如需修改：

- 在 `fcitx5-configtool` 中进入 “Global Config” 标签页
- 修改 “Trigger Input Method” 快捷键

---

## 验证效果

- 切换到 **Rime**：输入拼音（如 `nihao`）应输出简体中文（“你好”）
- 切换到 **Keyboard - English (US)**：直接输入英文字符，无候选窗、无延迟，适合终端或编程

---

## 常见问题排查

1. **无法呼出 Rime 菜单（Ctrl + `）** 确保已成功部署用户配置（`~/.local/share/fcitx5/rime/` 非空）
    
2. **英文模式仍有候选窗**  
    检查是否误添加了 Rime 的英文方案（如 `english`），应仅使用 “Keyboard - English (US)”
    
3. **系统未使用 Fcitx5**  
    进入 Ubuntu “Settings → Keyboard → Input Sources”，移除其他输入源，确保仅使用 “Fcitx 5”
    

---

此配置兼顾中文输入灵活性与英文输入的纯净性，适合日常使用及开发场景。