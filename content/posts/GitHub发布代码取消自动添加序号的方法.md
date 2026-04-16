# GitHub发布代码取消自动添加序号的方法

在GitHub发布Markdown文档时，代码前自动出现序号，核心原因主要有两种：一是代码块被误识别为「有序列表」，二是使用GitHub Pages（Jekyll等生成器）时全局开启了代码行号，以下分场景给出对应解决方法，适配日常Markdown文档（如README\.md）和GitHub Pages站点两种场景。

## 场景一：普通Markdown文档（README\.md等，非Pages站点）

此场景下序号多为「代码块误识别为有序列表」导致，GitHub的Markdown解析器（GFM）会将「数字\+\.\+空格\+代码」的格式误判为有序列表，进而自动排序添加序号，解决核心是规范代码块格式、避免解析冲突。

### 方法1：使用标准代码块语法（最常用、最稳妥）

用「三个反引号（\`\`\`）」包裹代码，指定代码语言（可选，不指定也可），即可避免被识别为列表，彻底取消序号。

**错误写法（易被误识别）**：未用反引号包裹，代码前有数字或空格，被误判为列表

```markdown
1. ls -l
2. cd /home
3. cat test.txt
```

**正确写法（取消序号）**：用反引号包裹，无论代码前有无数字，均不会生成序号

```bash
ls -l
cd /home
cat test.txt
```

说明：反引号包裹的代码块是GitHub GFM的标准语法，会被优先识别为代码，而非列表，从根源避免序号生成。

补充示例（对应你提供的grep命令，规范写法如下，可直接复制使用）：

```bash
grep "error" /var/log/messages  # 从系统日志中，筛选包含 error 的行
grep -n "root" /etc/passwd      # 查找 /etc/passwd 中包含 root 的行，并显示行号
grep -i "test" test.txt         # 查找 test.txt 中包含 test 的行，忽略大小写（匹配 Test、TEST 等）
grep -v "info" log.txt          # 查找 log.txt 中不包含 info 的所有行
```

注：上述写法用反引号包裹所有命令，即使命令前有数字（如你原始输入的1、2、3、4），也可删除数字后直接放入代码块，避免被识别为有序列表。

### 方法2：清除代码前的列表标记和多余缩进

若无需代码块高亮，仅需取消序号，可直接清除代码行前的「数字\+\.」（如1\.、2\.）和多余缩进，确保代码行无任何列表相关标记。

**错误写法**：代码前有列表标记或多余缩进

```markdown
1. which grep
  2. find . -name "*.log"
```

**正确写法**：删除数字\+\.和多余缩进，直接书写代码

```markdown
which grep
find . -name "*.log"
```

### 方法3：列表嵌套代码块（需保留列表，仅取消代码序号）

若需在有序/无序列表中插入代码块，需给代码块添加「8个空格」或「两个标准缩进」，确保代码块被识别为列表的子元素，而非独立列表项，避免自动排序。

**正确写法示例**：

```markdown
1. 查看系统用户命令
        getent passwd  # 8个空格缩进，代码不会被加序号
2. 修改文件权限命令
        chmod 755 test.txt
```

说明：缩进不足会导致代码块脱离列表、被误判为独立列表项，进而自动加序号，严格遵循8个空格缩进即可避免。

## 场景二：GitHub Pages站点（使用Jekyll等静态生成器）

若你发布的是GitHub Pages站点（如个人博客、项目文档站点），代码前的序号可能是Jekyll全局配置开启了「代码行号」导致，需修改站点配置文件关闭行号。

### 方法：修改Jekyll配置文件（\_config\.yml）

1\. 进入GitHub Pages站点的仓库，找到根目录下的 `\_config\.yml` 文件（无则新建）；

2\. 找到「kramdown」相关配置（Jekyll默认语法高亮引擎），添加/修改 `line\_numbers: false`，关闭全局代码行号；

**配置示例**：

```yaml
# _config.yml 中添加以下内容
kramdown:
  syntax_highlighter: rouge  # 语法高亮引擎（默认rouge）
  syntax_highlighter_opts:
    block:
      line_numbers: false    # 关闭代码块行号，取消自动序号
```

3\. 保存修改并提交到GitHub，等待站点重新构建后，代码前的序号会自动消失。

补充：若使用Liquid高亮标签（`\{% highlight %\}`），需确保标签后未添加 `linenos` 参数（该参数会强制显示行号），正确写法为 `\{% highlight bash %\}` 而非 `\{% highlight bash linenos %\}`。

## 常见问题排查

- 若按上述方法操作后仍有序号：检查代码块前后是否有隐藏的列表标记（如空格\+数字\+\.），可切换到GitHub仓库的「编辑模式」，查看原始Markdown文本，删除多余标记；

- 若仅部分代码有序号：大概率是部分代码块未用反引号包裹，或缩进不足，针对性修改对应代码块即可；

- GitHub直接查看代码文件（非Markdown文档）时显示的行号：属于GitHub默认功能，仅用于定位代码，并非Markdown渲染的序号，无法取消，不影响文档发布效果。

> （注：文档部分内容可能由 AI 生成）
