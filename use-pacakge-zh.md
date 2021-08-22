# `use-package`

[![Join the chat at https://gitter.im/use-package/Lobby](https://badges.gitter.im/use-package/Lobby.svg)](https://gitter.im/use-package/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![Build Status](https://travis-ci.org/jwiegley/use-package.svg?branch=master)](https://travis-ci.org/jwiegley/use-package)
[![MELPA](http://melpa.milkbox.net/packages/use-package-badge.svg)](http://melpa.milkbox.net/#/use-package)
[![MELPA Stable](https://stable.melpa.org/packages/use-package-badge.svg)](https://stable.melpa.org/#/use-package)

 `use-package` 宏允许你在你的 `.emacs` 文件中隔离包的配置，这种方式既注重性能，也很整洁。我创建它是因为我在Emacs中使用了80多个包，而且事情变得难以管理。 然而，有了这个工具，我的总加载时间大约是2秒，而且没有损失任何功能!

**NOTE**: `use-package`  **不是** 一个软件包管理器!  尽管 `use-package` 确实具有与包管理器对接的有用功能 (见[下文](#package-installation))，但它的主要目的是配置和加载包。

为升级到2.x的用户提供的说明位于 [底部](#upgrading-to-2x)。

## 安装 use-package

从这个 GitHub 存储库克隆或从 MELPA 安装(推荐)。

## 入门

这是最简单的 `use-package` 声明:

``` elisp
;; 这句只需一次，放在在文件的顶部。
(eval-when-compile
  ;; 如果use-package.el在~/.emacs.d中，则不需要以下一行
  (add-to-list 'load-path "<use-package 的安装路径>")
  (require 'use-package))

(use-package foo)
```

这将会加载软件包 `foo`, 前提是 `foo` 在你的系统上可用。否则会在 `*Messages*` buffer中记录警告.

使用 `:init` 关键字在包被加载之前执行代码。 它接受一个或多个forms，直到下一个关键字。

``` elisp
(use-package foo
  :init
  (setq foo-variable t))
```

类似的, `:config` 可以用来在一个包被加载后执行代码。如果设置了延迟加载(见下文关于 autoloading 的更多信息), 此操作将推迟到autoload发生之后：

``` elisp
(use-package foo
  :init
  (setq foo-variable t)
  :config
  (foo-mode 1))
```

如你所料, 你可以同时使用 `:init` 和 `:config` :

``` elisp
(use-package color-moccur
  :commands (isearch-moccur isearch-all)
  :bind (("M-s O" . moccur)
         :map isearch-mode-map
         ("M-o" . isearch-moccur)
         ("M-O" . isearch-moccur-all))
  :init
  (setq isearch-lazy-highlight t)
  :config
  (use-package moccur-edit))
```

此例中，我想从 `color-moccur.el`自动加载 `isearch-moccur` 和 `isearch-all` 命令, 并且在全局和 `isearch-mode-map` 内都绑定了快捷键，(见下一节).  当软件包实际被加载时(通过使用这些命令之一),  `moccur-edit` 也会被加载，以允许编辑 `moccur` 缓冲区。

## 按键绑定

在加载一个模块时，另一件常见的事情是将一个键绑定到该模块内的主要命令。

``` elisp
(use-package ace-jump-mode
  :bind ("C-." . ace-jump-mode))
```

这做了两件事：首先，它为 `ace-jump-mode` 命令创建了一个autoload，将 `ace-jump-mode` 的加载推迟到你实际使用它的时候。其次，它将 `C-.` 键与该命令绑定。在加载之后，你可以使用`M-x describe-personal-keybindings` 来查看您在整个 `.emacs` 中设置的所有此类键绑定。

更符合字面意思的方法是：

``` elisp
(use-package ace-jump-mode
  :commands ace-jump-mode
  :init
  (bind-key "C-." 'ace-jump-mode))
```

当你使用 `:commands` 关键字时，它会为这些命令创建autoload并将模块的加载推迟到使用它们之前。由于 `:init` 始终运行 -- 即使 `ace-jump-mode` 可能不在你的系统上 -- 记得限制:init代码，使其只在任何情况下都能执行成功。

 `:bind` 关键字可以接受一个 cons 或一个 cons 列表。

``` elisp
(use-package hi-lock
  :bind (("M-o l" . highlight-lines-matching-regexp)
         ("M-o r" . highlight-regexp)
         ("M-o w" . highlight-phrase)))
```

 `:commands` 关键字同样接受一个符号或一个符号列表。

**NOTE**: 在字符串中，特殊键如 `tab` 或 `F1`-`Fn` 必须写在角括号内，例如 `"C-<up>"`. 独立的特殊键（和一些组合）可以写在方括号内，例如 `[tab]` 而不是 `"<tab>"`. 键绑定的语法与 "kbd "语法类似: 见 [https://www.gnu.org/software/emacs/manual/html_node/emacs/Init-Rebinding.html](https://www.gnu.org/software/emacs/manual/html_node/emacs/Init-Rebinding.html) 以了解更多信息。

例子:

``` elisp
(use-package helm
  :bind (("M-x" . helm-M-x)
         ("M-<f5>" . helm-find-files)
         ([f10] . helm-buffers-list)
         ([S-f10] . helm-recentf)))
```

此外,带有 `:bind` 和 `bind-key` 的 [remapping commands](https://www.gnu.org/software/emacs/manual/html_node/elisp/Remapping-Commands.html) 如预期的那样工作，因为当绑定是一个矢量时，它被直接传递给 `define-key`。所以下面的例子将把 `M-q` (原来是 `fill-paragraph`) 重新绑定到 
`unfill-toggle`:

``` elisp
(use-package unfill
  :bind ([remap fill-paragraph] . unfill-toggle))
```

### 绑定到keymaps

通常 `:bind` 期望命令是将从给定包自动加载的函数。 然而，如果这些命令之一实际上是keymap，这就不起作用了，因为keymap不是函数，并且不能使用 Emacs 的 `autoload` 机制自动加载。

为了处理这种情况， `use-package` 提供了一个特殊的、受限的 `:bind` 变体，叫做 `:bind-keymap`. 唯一的区别是，由 `:bind-keymap` 绑定的 "命令 "必须是包中定义的keymap，而不是命令函数。这是通过生成自定义代码在后台处理的，该代码会加载包含keymap的包，然后在第一次加载后重新执行你的按键，将该按键重新解释为前缀键。

比如说：

``` elisp
(use-package projectile
  :bind-keymap
  ("C-c p" . projectile-command-map))
```

### 局部keymaps绑定

与将键绑定到keymap略有不同的是，在局部keymap中绑定一个键，该keymap仅在加载包后才存在。 `use-package`通过 `:map` 修饰符支持这一点，将局部keymap绑定到：

``` elisp
(use-package helm
  :bind (:map helm-command-map
         ("C-c h" . helm-execute-persistent-action)))
```

这条语句的作用是等待 `helm` 加载完毕，然后将 `C-c h` 键与 `helm-execute-persistent-action` 绑定在Helm的局部keymap `helm-mode-map`中。

可以指定多次使用 `:map` 。在第一次使用 `:map` 之前发生的任何绑定都会应用于全局keymap:

``` elisp
(use-package term
  :bind (("C-c t" . term)
         :map term-mode-map
         ("M-p" . term-send-up)
         ("M-n" . term-send-down)
         :map term-raw-map
         ("M-o" . other-window)
         ("M-p" . term-send-up)
         ("M-n" . term-send-down)))
```

## 模式和解释器

与 `:bind` 类似, 你可以使用 `:mode` 和 `:interpreter` 在 `auto-mode-alist` 和 `interpreter-mode-alist`变量中建立延迟绑定。这两个关键字的参数可以是 cons cell、cons cell 列表、字符串或正则表达式：

``` elisp
(use-package ruby-mode
  :mode "\\.rb\\'"
  :interpreter "ruby")

;; 包名为 "python" 但模式名为 "python-mode":
(use-package python
  :mode ("\\.py\\'" . python-mode)
  :interpreter ("python" . python-mode))
```

如果你不使用 `:commands`, `:bind`, `:bind*`, `:bind-keymap`, `:bind-keymap*`, `:mode`, `:interpreter`, 或 `:hook` (所有这些都意味着 `:defer`; 有关它们的简要说明，参阅 `use-package` 的文档字符串), 你仍然可以用 `:defer` 关键字来延迟加载。

``` elisp
(use-package ace-jump-mode
  :defer t
  :init
  (autoload 'ace-jump-mode "ace-jump-mode" nil t)
  (bind-key "C-." 'ace-jump-mode))
```

这与下面的做法完全相同：

``` elisp
(use-package ace-jump-mode
  :bind ("C-." . ace-jump-mode))
```

## Magic handlers

与 `:mode` 和 `:interpreter` 类似，如果文件的开头与给定的正则表达式匹配，你也可以使用  `:magic` 和 `:magic-fallback` 使某些函数运行。两者的区别在于 `:magic-fallback` 的优先级低于 `:mode`. 例如:

``` elisp
(use-package pdf-tools
  :load-path "site-lisp/pdf-tools/lisp"
  :magic ("%PDF" . pdf-view-mode)
  :config
  (pdf-tools-install :no-query))
```

这会为 `pdf-view-mode` 注册一个autoload的命令，推迟 `pdf-tools`的加载，并在buffer的开头与字符串 `"%PDF"` 匹配时运行 `pdf-view-mode` .

## Hooks

`:hook` 关键字允许将函数添加到包的钩子上。 因此，以下所有内容都是等价的：

``` elisp
(use-package ace-jump-mode
  :hook prog-mode)

(use-package ace-jump-mode
  :hook (prog-mode . ace-jump-mode))

(use-package ace-jump-mode
  :commands ace-jump-mode
  :init
  (add-hook 'prog-mode-hook #'ace-jump-mode))
```

同样，当应该应用多个钩子时，以下也是等效的：

``` elisp
(use-package ace-jump-mode
  :hook (prog-mode text-mode))

(use-package ace-jump-mode
  :hook ((prog-mode text-mode) . ace-jump-mode))

(use-package ace-jump-mode
  :hook ((prog-mode . ace-jump-mode)
         (text-mode . ace-jump-mode)))

(use-package ace-jump-mode
  :commands ace-jump-mode
  :init
  (add-hook 'prog-mode-hook #'ace-jump-mode)
  (add-hook 'text-mode-hook #'ace-jump-mode))
```

使用 `:hook` 时如果您明确指定了钩子，请省略“-hook”后缀，因为这是默认附加的。 例如，以下代码将不起作用，因为它尝试添加到不存在的 `prog-mode-hook-hook`:

``` elisp
;; 不起作用
(use-package ace-jump-mode
  :hook (prog-mode-hook . ace-jump-mode))
```

如果你不喜欢这种行为，请将 `use-package-hook-name-suffix` 设置为nil。默认情况下，这个变量的值是"-hook"。

 `:hook` 的使用，与 `:bind`, `:mode`, `:interpreter`等一样, 会导致被钩住的函数被隐含地读取为 `:commands` (这意味着它们将为该模块建立交互式 `autoload` 定义, 如果尚未定义为函数的话), 因此 `:defer t` 也被 `:hook`所隐含。

## 软件包自定义

### 自定义variables

 `:custom` 关键字允许对包的自定义变量进行定制。

``` elisp
(use-package comint
  :custom
  (comint-buffer-maximum-size 20000 "Increase comint buffer size.")
  (comint-prompt-read-only t "Make the prompt read only."))
```

字符串文档不是强制性的。

**NOTE**: 这些仅适用于希望通过随附的 usepackage 声明保留自定义项的人。 在功能上，在 `:config` 块中使用 `setq`  的唯一好处是自定义可能会在分配值时执行代码。

**NOTE**: 自定义值**不会**保存在 Emacs `custom-file`.因此，您应该使用 `:custom` 选项 **或者** 您应该使用 `M-x customize-option` ，它将把自定义的值保存在Emacs `custom-file`.不要同时使用两者。

### 自定义faces

 `:custom-face` 关键字允许定制包的自定义faces.

``` elisp
(use-package eruby-mode
  :custom-face
  (eruby-standard-face ((t (:slant italic)))))
```

## 关于延迟加载的注意事项

在几乎所有情况下，你都不需要手动指定 `:defer t`.  每当使用 `:bind` 、 `:mode` 或 `:interpreter` 时，这都是隐含的。通常情况下，只有当你知道某个其他软件包会在适当的时候执行某些操作以导致此软件包被加载时，你才需要指定 `:defer` ，因此，即使 use-package 没有为您创建任何自动加载，您也希望推迟加载。

你可以用 `:demand` 关键字来取消包的延迟加载。于是，即使你使用了 `:bind`， `:demand` 也会强制立即加载软件包。

## 关于包的加载

当一个包被加载时，如果你把 `use-package-verbose` 设置为 `t`, 或者包的加载时间超过0.1s，你会在 `*Messages*` buffer中看到一条信息来表明这个加载活动。配置或 `:config` 块的执行时间超过0.1s时也会发生同样的情况。一般来说，你应该让 `:init` 尽可能的简单和快速，并且尽量把东西放入 `:config` 块中。这样，延迟加载可以帮助您的 Emacs 尽快启动。

此外，如果在初始化或配置软件包时发生错误，这不会使你的Emacs停止加载。相反，错误将由 `use-package` 捕获，并报告给一个特殊的 `*Warnings*`  popupbuffer，以便您可以在其他功能的 Emacs 中调试这种情况。

## 条件加载

你可以使用 `:if` 关键字来预测模块的加载和初始化。

例如，我只想让 `edit-server` 为我的GUI Emacs运行, 而不是为我可能在命令行启动的其他Emacs运行:

``` elisp
(use-package edit-server
  :if window-system
  :init
  (add-hook 'after-init-hook 'server-start t)
  (add-hook 'after-init-hook 'edit-server-start t))
```
在另一个例子中，我们可以以操作系统为条件加载东西。

``` elisp
(use-package exec-path-from-shell
  :if (memq window-system '(mac ns))
  :ensure t
  :config
  (exec-path-from-shell-initialize))
```

 `:disabled` 关键字可以关闭一个你难以使用的模块。或停止加载你目前不使用的东西。

``` elisp
(use-package ess-site
  :disabled
  :commands R)
```

在对 `.emacs` 文件进行字节编译时，输出中完全省略了禁用声明，以加快启动时间。

**NOTE**: `:when` 是作为 `:if ` 的别名提供的，而 `:unless foo` 与 `:if (not foo)` 意思相同。例如，下面的内容也会阻止 `:ensure` 在Mac系统上发生:

``` elisp
(when (memq window-system '(mac ns))
  (use-package exec-path-from-shell
    :ensure t
    :config
    (exec-path-from-shell-initialize)))
```

### :preface 之前的条件加载

如果您需要对 use-package form进行条件化，以便该条件甚至在 `:preface` 执行之前发生，只需在 use-package form本身周围使用 `when` :

### 按顺序加载软件包

有时，只有在另一个包被加载后配置此包才有意义，因为某些变量或函数直到那时才在范围内。这可以使用 `:after` 关键字来实现，该关键字允许对应该发生加载的确切条件进行相当丰富的描述。 下面是一个例子：

``` elisp
(use-package hydra
  :load-path "site-lisp/hydra")

(use-package ivy
  :load-path "site-lisp/swiper")

(use-package ivy-hydra
  :after (ivy hydra))
```

在这种情况下，因为所有这些包都是按照它们出现的顺序按需加载的，所以 `:after` 的使用不是绝对必要的。 然而，通过使用它，上面的代码变得与顺序无关，对 init 文件的性质没有隐含的依赖。

默认情况下， `:after (foo bar)` 与 `:after (:all foo bar)` 相同，这意味着在 `foo` 和 `bar` 被加载之前不会加载给定的包。 以下是其他一些可能性：

``` elisp
:after (foo bar)
:after (:all foo bar)
:after (:any foo bar)
:after (:all (:any foo bar) (:any baz quux))
:after (:any (:all foo bar) (:all baz quux))
```

当你嵌套筛选器时，例如 `(:any (:all foo bar) (:all baz quux))`，这意味着当 `foo` 和 `bar` 都已加载，或者 `baz` 和 `quux` 都已加载时，该包将被加载。

**NOTE**: 如果你把 `use-package-always-defer` 设置为 t, 并且还使用了 `:after` 关键字，那么请注意，你将需要指定声明的包如何被加载：例如，通过设置一些 `:bind`。如果你没有使用注册自动加载的机制之一，比如 `:bind` or `:hook` ，而且你的包管理器不提供自动加载，那么如果不在这些声明中加入  `:demand t` ，你的包就有可能永远不会被加载。

### 如果缺少依赖项，则阻止加载

虽然 `:after` 关键字会延迟加载，直到依赖项被加载为止，但稍微简单的 `:requires` 关键字如果在遇到 `use-package` 声明时依赖项不可用，则永远不会加载包。这里所说的 "可用 "是指如果 `(featurep 'foo)` 被判定为 non-nil，则  `foo` 就是可用的。比如说：

``` elisp
(use-package abbrev
  :requires foo)
```

这就相当于：

``` elisp
(use-package abbrev
  :if (featurep 'foo))
```

为了方便起见，可以指定一个这样的软件包列表：

``` elisp
(use-package abbrev
  :requires (foo bar baz))
```

对于更复杂的逻辑，例如 `:after` 支持的逻辑，只需使用 `:if` 和适当的 Lisp 表达式。

## 字节编译你的 .emacs

 `use-package` 的另一个特点是它总是在 `.emacs` 被字节编译时加载它可以加载的每个文件。 这有助于消除有关未知变量和函数的虚假警告。

然而，有时这还不够。在这种情况下，请使用 `:defines` 和 `:functions` 关键字来引入虚拟变量和函数声明，这完全是为了字节编译器的需要：

``` elisp
(use-package texinfo
  :defines texinfo-section-list
  :commands texinfo-mode
  :init
  (add-to-list 'auto-mode-alist '("\\.texi$" . texinfo-mode)))
```

如果您需要消除缺少函数的警告，您可以使用 `:functions`:

``` elisp
(use-package ruby-mode
  :mode "\\.rb\\'"
  :interpreter "ruby"
  :functions inf-ruby-keys
  :config
  (defun my-ruby-mode-hook ()
    (require 'inf-ruby)
    (inf-ruby-keys))

  (add-hook 'ruby-mode-hook 'my-ruby-mode-hook))
```

### 防止一个软件包在编译时被加载

通常， `use-package` 在编译配置之前会在编译时加载每个包，以确保任何必要的符号都在范围内以满足字节编译器。有时这可能会导致问题，因为某个包可能有特殊的加载要求，而你使用 `use-package` 的目的只是向 `eval-after-load` hook添加一个配置。 在这种情况下，请使用 `:no-require` 关键字:

``` elisp
(use-package foo
  :no-require t
  :config
  (message "这句用来判定foo被加载了"))
```

## 扩展load-path

如果你的软件包需要一个添加到 `load-path` 中的目录才能加载，请使用 `:load-path`.  如果路径是相对的，它会在 `user-emacs-directory` 中展开:

``` elisp
(use-package ess-site
  :load-path "site-lisp/ess/lisp/"
  :commands R)
```

**NOTE**: 当使用符号或函数来提供动态生成的路径列表时，您必须将此定义通知字节编译器，以便该值在字节编译时可用。 这是通过使用特殊形式 `eval-and-compile` (与 `eval-when-compile` 相反)来完成的。此外，该值固定为编译期间确定的任何值，以避免在每次启动时再次查找相同的信息：

``` elisp
(eval-and-compile
  (defun ess-site-load-path ()
    (shell-command "find ~ -path ess/lisp")))

(use-package ess-site
  :load-path (lambda () (list (ess-site-load-path)))
  :commands R)
```

## 抓取use-package扩展过程中的错误

默认情况下，如果 `use-package-expand-minimally` 为 nil (默认值)，use-package 将尝试捕获并报告在 init 文件中扩展 use-package 声明时发生的错误。将 `use-package-expand-minimally` 设置为 t 将完全禁用此检查。

可以使用 `:catch` 关键字在本地覆盖此行为。如 `t` 或 `nil` ，它启用或禁用在加载时捕获错误。它也可以是一个带有两个参数的函数：遇到错误时正在处理的关键字，以及错误对象 (由 `condition-case` 生成)。 例如：

``` elisp
(use-package example
  ;; Note 错误永远不会被困在前言中，因为这样做会使定义不被字节编译器发现。
  :preface (message "I'm here at byte-compile and load time.")
  :init (message "I'm always here at startup")
  :config
  (message "I'm always here after the package is loaded")
  (error "oops")
  ;; 不要尝试 (require 'example), 这只是一个示例!
  :no-require t
  :catch (lambda (keyword err)
           (message (error-message-string err))))
```

Evaluating上述form将打印以下消息:

```
I’m here at byte-compile and load time.
I’m always here at startup
Configuring package example...
I’m always here after the package is loaded
oops
```

## Diminishing和delighting辅模式

`use-package` 还提供了对 diminish 和 delight 实用程序的内置支持 -- 前提是你安装了它们。它们的目的是删除或更改 mode-line中的辅模式字符串。

[diminish](https://github.com/myrjola/diminish.el) 是通过 `:diminish` 关键字调用的，该关键字传递一个辅模式符号，一个包含符号及其替换字符串的cons，或者只是一个替换字符串，在这种情况下，辅模式符号被猜测为在末尾附加了 "-mode" 的包名称:

``` elisp
(use-package abbrev
  :diminish abbrev-mode
  :config
  (if (file-exists-p abbrev-file-name)
      (quietly-read-abbrev-file)))
```

[delight](https://elpa.gnu.org/packages/delight.html) 是通过 `:delight` 关键字调用的，该关键字接收一个辅模式符号、一个替换字符串或带引号的
[mode-line data](https://www.gnu.org/software/emacs/manual/html_node/elisp/Mode-Line-Data.html) (在这种情况下，次要模式符号被猜测为在末尾附加"-mode"的包名)，这两者，或两者的几个列表。 如果未提供参数，默认的模式名称将被完全隐藏。

``` elisp
;; 不要为rainbow-mode显示任何内容。
(use-package rainbow-mode
  :delight)

;; 不要为 auto-revert-mode 显示任何东西，这与它的软件包名称不匹配。
(use-package autorevert
  :delight auto-revert-mode)

;; 删除 projectile-mode 的模式名称，但显示项目名称。
(use-package projectile
  :delight '(:eval (concat " " (projectile-project-name))))

;; 完全隐藏 visual-line-mode 并将 auto-fill-mode 更改为 " AF".
(use-package emacs
  :delight
  (auto-fill-function " AF")
  (visual-line-mode))
```

## 软件包安装

你可以使 `use-package` 调用 `package.el` 从 ELPA 加载包。如果您在多台机器之间共享您的 `.emacs` ，这将特别有用； 一旦在您的 `.emacs` 中声明，相关的包就会自动下载。如果你的系统中还没有软件包，那么 `:ensure` 关键字将使软件包自动安装。

``` elisp
(use-package magit
  :ensure t)
```

如果您需要安装一个与 `use-package` 命名不同的软件包，您可以像这样指定它：

``` elisp
(use-package tex
  :ensure auctex)
```

如果你希望对所有软件包都这样处理，请启用 `use-package-always-ensure` ：

``` elisp
(require 'use-package-ensure)
(setq use-package-always-ensure t)
```

**NOTE**: `:ensure` 将安装尚未安装的软件包，但不会使其保持最新状态。如果你想让你的包自动更新，一种选择是使用 [auto-package-update](https://github.com/rranelli/auto-package-update.el),比如

``` elisp
(use-package auto-package-update
  :config
  (setq auto-package-update-delete-old-versions t)
  (setq auto-package-update-hide-results t)
  (auto-package-update-maybe))
```

最后，在 Emacs 24.4 或更高版本上运行时，use-package 可以将包固定到特定的仓库，允许您混合和匹配来自不同仓库的包。 主要用例是更喜欢来自 `melpa-stable` 和 `gnu` 仓库的包，但是当您需要跟踪比 `stable` 仓库中可用的版本更新的版本时，使用来自 `melpa` 的特定包也是一个有效的用例。

默认情况下， `package.el` 由于版本的原因，更倾向于 `melpa` 而非 `melpa-stable`  `(> evil-20141208.623 evil-1.0.9)`，因此即使您只跟踪来自 `melpa` 的单个包，你也需要用相应的仓库来标记所有非`melpa`的包。如果这真的让你恼火，那么你可以用 `use-package-always-pin` 来设置一个默认值。

如果您想手动保持包更新并忽略上游更新，您可以将其固定为 `manual`，只要没有同名的存储库，就可以正常工作。

如果您尝试将包固定到尚未使用 `package-archives` 配置的仓库， `use-package` 会产生一个错误。(除了上面提到的神奇的 `manual` 仓库):

```
Archive 'foo' requested for package 'bar' is not available.
```

示例:

``` elisp
(use-package company
  :ensure t
  :pin melpa-stable)

(use-package evil
  :ensure t)
  ;; 不使用 :pin, package.el将选择melpa中的版本

(use-package adaptive-wrap
  :ensure t
  ;; 由于这个包只在gnu档案中提供，因此在技术上不需要这样做，但它有助于突出显示它的来源
  :pin gnu)

(use-package org
  :ensure t
  ;; 忽略上游的org-mode，使用手动安装的版本
  :pin manual)
```

**NOTE**: `:pin` 参数在< 24.4版本的emacs中不起作用。

### 与其他包管理器配合使用

通过覆盖 `use-package-ensure-function` and/or `use-package-pre-ensure-function`, 其他包管理器可以覆盖 `:ensure` 来使用它们而不是 `package.el`. 目前，唯一能做到这一点的软件包管理器是 [`straight.el`](https://github.com/raxod502/straight.el).

## 收集统计数据

如果您想查看您加载了多少个包，它们到达了初始化的哪个阶段，以及它们总共花费了多少时间(大致)，您可以在加载 `use-package` 之后但在所有 `use-package` form之前启用 `use-package-compute-statistics` ，然后使用命令 `M-x use-package-report` 来查看结果。显示的buffer是一个表格列表。 您可以在列中使用 `S` 根据它对行进行排序。

## 关键字扩展

从 2.0 版本开始， `use-package` 基于一个可扩展的框架，使包作者可以轻松地添加新关键字或修改现有关键字的行为。

一些关键字扩展现在已经包含在 `use-package` 分发中，可以选择安装。

### `(use-package-ensure-system-package)`

 `:ensure-system-package` 关键字允许您确保系统二进制文件与包声明一起存在。

首先，你要确保 `exec-path` 知道你想要确保已安装的所有二进制包名称。 [`exec-path-from-shell`](https://github.com/purcell/exec-path-from-shell)
通常是执行此操作的好方法。

要在加载 `use-package` 后启用扩展:

``` elisp
(use-package use-package-ensure-system-package
  :ensure t)
```

下面是一个使用示例：

``` emacs-lisp
(use-package rg
  :ensure-system-package rg)
```

这将期望存在一个名为 `rg`的全局二进制包。如果没有，它将使用您的系统包管理器 (使用 [`system-packages`](https://gitlab.com/jabranham/system-packages)包) 尝试异步安装同名的二进制文件。例如，对于大多数 `macOS` 用户，这将调用：`brew install rg`.

如果包的名称与二进制文件不同，则可以使用  `(binary . package-name)`形式的 cons，即：

``` emacs-lisp
(use-package rg
  :ensure-system-package
  (rg . ripgrep))
```

在上面的 `macOS` 示例中，如果未找到 `rg` ,将会调用 `brew install
ripgrep` .

如果你想进一步自定义安装命令怎么办？

``` emacs-lisp
(use-package tern
  :ensure-system-package (tern . "npm i -g tern"))
```

`:ensure-system-package` 也可以接受一个 cons ，它的 `cdr` 是一个字符串，如果没有找到它，它将被 `(async-shell-command)` 调用以进行安装。

你也可以传入一个cons列表:

``` emacs-lisp
(use-package ruby-mode
  :ensure-system-package
  ((rubocop     . "gem install rubocop")
   (ruby-lint   . "gem install ruby-lint")
   (ripper-tags . "gem install ripper-tags")
   (pry         . "gem install pry")))
```

最后，如果包依赖项不提供全局可执行文件，您可以通过提供如下字符串来检查文件路径的存在以确保包存在：

``` emacs-lisp
(use-package dash-at-point
  :if (eq system-type 'darwin)
  :ensure-system-package
  ("/Applications/Dash.app" . "brew cask install dash"))
```

`:ensure-system-package` 将使用 `system-packages-install` 来安装系统包，除非指定了自定义命令，在这种情况下，它将由 `async-shell-command` 逐字执行。 

配置变量 `system-packages-package-manager` 和 `system-packages-use-sudo` 将受到尊重，但不适用于自定义命令。 如果需要，自定义命令应在命令中包含对 sudo 的调用。

### `(use-package-chords)`

 `:chords` 关键字允许您以与 `:bind` 关键字相同的方式为 `use-package` 的声明定义一个[`key-chord`](http://www.emacswiki.org/emacs/key-chord.el)绑定。

要启用扩展：

``` elisp
(use-package use-package-chords
  :ensure t
  :config (key-chord-mode 1))
```

然后，您可以用和 `:bind` 相同的方式，用一个cons或cons列表定义chord绑定

``` elisp
(use-package ace-jump-mode
  :chords (("jj" . ace-jump-char-mode)
           ("jk" . ace-jump-word-mode)
           ("jl" . ace-jump-line-mode)))
```

### 如何创建扩展

#### 第一步：添加关键字

第一步是在
`use-package-keywords`中的正确位置添加你的关键字。 这个列表决定了事情在扩展代码中发生的顺序。 您永远不应该更改此顺序，但它为你提供了一个框架，你可以在其中决定何时触发关键字。

#### 第二步：创建normalizer

通过定义一个以关键字命名的函数，为你的关键词定义一个normalizer，比如说：

``` elisp
(defun use-package-normalize/:pin (name-symbol keyword args)
  (use-package-only-one (symbol-name keyword) args
    (lambda (label arg)
      (cond
       ((stringp arg) arg)
       ((symbolp arg) (symbol-name arg))
       (t
        (use-package-error
         ":pin wants an archive name (a string)"))))))
```

normalizer的作用是接收一个参数列表 (可能是nil)，并将其转化为单个参数 (它仍然可以是一个列表)，该参数应该出现在 `use-package` 使用的最终属性列表中。

#### 第三步：创建handler

一旦有了normalizer，就必须为关键字创建一个handler：

``` elisp
(defun use-package-handler/:pin (name-symbol keyword archive-name rest state)
  (let ((body (use-package-process-keywords name-symbol rest state)))
    ;; 这发生在宏扩展时，而不是在编译或评估扩展代码时。
    (if (null archive-name)
        body
      (use-package-pin-package name-symbol archive-name)
      (use-package-concat
       body
       `((push '(,name-symbol . ,archive-name)
               package-pinned-packages))))))
```

Handlers 可以通过两种方式影响对关键词的处理。首先，它可以在递归处理剩余关键字之前修改 `state` plist 以影响关注state的关键字 (例如state关键字 `:deferred`, 不要与 `use-package` 关键字 `:defer` 混淆)。然后，一旦处理完剩余的关键字并返回它们的结果forms，handler就可以操作、扩展或直接忽略这些forms。

每个处理程序的任务是返回一个表示要插入的代码的表格列表。他不必是 `progn` 列表，因为这在其他地方会自动处理。因此，使用 `use-package-concat` 在代码体之前或之后添加新的功能是非常常见的，因此，只有最低限度的必要代码才会作为 `use-package` 扩展的结果被释放出来。

#### 第四步：最终测试

在关键字被插入到 `use-package-keywords` 中，并且定义了normalizer和handler之后，你现在可以通过查看关键字的使用情况来测试它。为此，请使用 `M-x pp-macroexpand-last-sexp` 并将光标设置在 `(use-package ...)` 表达式之后。

## 一些计时结果

在我的 Retina iMac上，Emacs 24.4的 "Mac port" 变体在0.57秒内加载完毕，配置了大约218个软件包 (几乎所有包都是延迟加载的)。然而，我没有遇到任何功能损失，只是在我第一次启动Emacs时有一点延迟 (由于autoloading)。由于我对许多软件包也使用了idle-loading
for，所以总体而言，感知到的延迟通常会减少。

在 Linux 上，相同的配置在0.32秒内加载完毕。

如果我不以图形方式使用 Emacs，我可以通过运行以下代码来测试最小绝对时间：

``` bash
time emacs -l init.elc -batch --eval '(message "Hello, world!")'
```

在 Mac 上，我看到相同配置的平均为 0.36秒，而在 Linux 上为 0.26秒。

# 升级到2.x

## :init 的语义现在是一致的

 `:init` 的含义已经改变：它现在总是发生在软件包加载之前，无论 `:config` 是否被延迟。这意味着在你的配置中，一些对 `:init` 的使用可能需要改为 `:config`(在非延迟的情况下)。对于延迟的情况，其行为与之前没有变化。

另外，因为 `:init` and `:config` 现在的意思是“之前”和“之后”，所以 `:pre-`和 `:post-` 关键字已经消失，因为它们不再是必需的。

最后，我们努力使你的Emacs在 use-package 配置失败的情况下也能启动。所以在这个改变之后，一定要检查你的 `*Messages*` buffer。 最有可能的是，您将有几个使用 `:init` 的实例，但其实应该使用 `:config` (我在很多地方都是这种情况)。

## :idle 已被删除

我暂时删除了这个功能，因为它可能会导致严重的不一致。 考虑以下定义：

``` elisp
(use-package vkill
  :commands vkill
  :idle (some-important-configuration-here)
  :bind ("C-x L" . vkill-and-helm-occur)
  :init
  (defun vkill-and-helm-occur ()
    (interactive)
    (vkill)
    (call-interactively #'helm-occur))

  :config
  (setq vkill-show-all-processes t))
```

如果我加载我的 Emacs 并等到idle计时器启动，那么事件的顺序是这样的：

    :init :idle <load> :config

但如果我加载Emacs并立即输入C-x L，而不等待idle计时器启动，这就是事件的顺序。

    :init <load> :config :idle

用户有可能在他们的idle中使用 `featurep` 来测试这种情况，但这是一个我宁愿避免的微妙问题。

## :defer 现在接受一个可选的数字参数

`:defer [N]` 会使一个还没有被加载的软件在 `N` 秒的空闲时间之后被加载。

``` elisp
(use-package back-button
  :commands (back-button-mode)
  :defer 2
  :init
  (setq back-button-show-toolbar-buttons nil)
  :config
  (back-button-mode 1))
```

## 添加 :preface，出现在除 :disabled之外的所有内容之前

`:preface` 可用于建立函数和变量定义，这些定义将 1) 让字节编译器满意(它不会为那些定义不明的函数报错，因为你把它们放在一个保护块中)。2)允许你定义可在 `:if` 测试中使用的代码。

**NOTE**: 为了确保Lisp评估器和字节编译器都能看到定义，在 `:preface` 中指定的任何内容都会在加载时和字节编译时被评估，因此您应该避免在preface中产生任何副作用 ，并将其限制为符号声明和定义。

## 添加 :functions，用于向字节编译器声明函数

 `:defines` 用于变量， `:functions` 用于函数。

## 运行时不再需要use-package.el

这意味着您应该将以下内容放在 Emacs 的顶部，以进一步减少加载时间：

``` elisp
(eval-when-compile
  (require 'use-package))
(require 'diminish)                ;; 如果你使用 :diminish
(require 'bind-key)                ;; 如果你使用任何 :bind变体
```
