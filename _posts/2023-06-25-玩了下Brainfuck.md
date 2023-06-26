---
title: 用Lua写了个Brainfuck解释器后受了一夜苦
date: 2023-06-25 22:59:54 +08:00
tags: Lua Brainfuck
excerpt: 久闻Brainfuck大名，上周了解图灵完备的时候又看到了这个语言，便心血来潮用Lua实现了一个，受了一夜苦。
license: CC BY 4.0
---

久闻Brainfuck大名，最早是在SILI[^SILI]的功能里得知的。上周了解图灵完备的时候又看到了这个语言[^Turing]，便心血来潮用Lua实现了一个。

Brainfuck假设有一个长度至少为30,000的一维数组，每个单元为1字节。初始状态所有单元都为0，数据指针指向首个单元，指令指针指向首个指令。8个指令也很简单：

| 字符 | 作用 |
| --- | --- |
| `<` / `>` | 数据指针左移/右移一个单元。 |
| `+` / `-` | 当前单元的值加1/减1。 |
| `,` | 读取输入的一个字节，存入当前单元。 |
| `.` | 输出当前单元的值（当作ASCII码）。 |
| `[` | 当前单元的值不为0时，执行下一条指令；否则跳转到与之配对的`]`之后。 |
| `]` | 指令指针回到与之配对的`[`的位置。实际与“当前单元的值不为0时再跳转”等价[^enwp]。 |
{:.nowrap-first}

这8个字符以外的字符会被忽略。

在Lua实现方面，自然是用表来充当数组，不过并未真的初始化3万多个单元，而是配置了`__index`元表给不存在的键返回0。

```lua
--[[
 - Brainfuck解释器
 -
 - 作者：amero
 - 协议：MIT License
 -]]

local Brainfuck = {}

local insert = table.insert
local push = insert
local remove = table.remove
local pop = remove

--[[
 - commands: string
 - input: string | nil
 -]]
function Brainfuck.main(commands, input)
	-- 数据
	local cells = setmetatable({}, {
			__index = function(t, k) return 0 end,
		})
	local dp  -- data pointer
	local input_buffer
	local output_buffer

	-- 指令
	local instructions
	local bracket_pair_index
	local ip  -- instruction pointer
	local commands_len
	local command_funcs

	instructions = {
		['>'] = function() dp = dp + 1 end,

		['<'] = function() dp = dp - 1 end,

		['+'] = function()
			cells[dp] = (cells[dp] + 1) % 256
		end,

		['-'] = function()
			cells[dp] = (cells[dp] - 1) % 256
		end,

		['.'] = function()
			insert(output_buffer, cells[dp])
		end,

		[','] = function()
			cells[dp] = remove(input_buffer, 1)
		end,

		['['] = function()
			if cells[dp] == 0 then
				ip = bracket_pair_index[ip]
			end
		end,

		[']'] = function()
			if cells[dp] ~= 0 then
				ip = bracket_pair_index[ip]
			end
		end,
	}

	-- 预处理代码。创建成对的括号索引；将字符串转换为函数序列
	commands = string.gsub(commands, '[^><%+%-%.,%[%]]', '')
	command_funcs = {}
	bracket_pair_index = {}
	do
		local i = 1
		local bracket_stack = {}
		for char in string.gmatch(commands, '.') do
			command_funcs[i] = instructions[char]

			-- 寻找'['对应的']'，并创建索引
			if char == '[' then
				push(bracket_stack, i)
			elseif char == ']' then
				assert(#bracket_stack > 0, '方括号不配对')
				local left = pop(bracket_stack)
				bracket_pair_index[left] = i
				bracket_pair_index[i] = left
			end

			i = i + 1
		end
		assert(#bracket_stack == 0, '方括号不配对')
		commands_len = i - 1
	end

	-- 运行
	ip, dp = 1, 1
	input_buffer = input and { string.byte(input, 1, #input) } or {}
	insert(input_buffer, 0xFF)  -- EOF
	output_buffer = {}

	while ip <= commands_len do
		command_funcs[ip]()
		ip = ip + 1
	end

	return string.char(unpack(output_buffer))
end

return Brainfuck
```

写解释器并没有花多长时间——大的还在后面。

在Brainfuck中，若想给一个单元赋值，你当然可以写一大堆加号或减号，但很多时候数值很大，做乘法是个更好的选择。例如，数据指针在单元0，要令单元1等于73，你可以直接写73个加号：

```brainfuck
>+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

也可以凑73 = 8 × 9 + 1，然后进行8次循环，每次给单元1加9，循环结束后给单元1加1：

```brainfuck
++++++++[>+++++++++<-]>+
```

我用Brainfuck写了一个把ASCII字母大写转小写的程序，光是这就花了我一夜，太阴间了。

- 想要复制一个单元的值到其他单元，就得牺牲值所在的单元。正如这个程序中的注释“复制`charIn`到`charCpy`和`charOut`中”所对应的指令，执行完后`charIn`单元也就为0了~~，燃烧自己，照亮他人~~。
- 想用`if`？行，但做好作为条件的单元被清空的准备，不然是无法跳出循环的。
- ……

```brainfuck
[由于内存单元初始是0，这对方括号里的代码（+-<>[],.）会被跳过，所以可以当作文档注释。
该程序实现了将输入的内容中的大写字母转换为对应小写字母，而其它字符不变。]

MEM: zero zero charIn
     ^ 0    0     0
>>(charIn),+[-                          读取字符，当字符不是EOF时进入循环
    将charIn减'A'，并复制为两份：charCpy在下一段用，charOut在最终输出的时候用。
    MEM: charOut charCpy charIn counter
            0       0     ^ X      0
    >(counter)++++++++[<-------->-]<-   charIn减'A'的ASCII码65 (8 * 负8 减 1)
    (charIn)[<+<+>>-]                   复制charIn到charCpy和charOut中

    判断是charCpy否小于等于26，若是，则isLT26非0。
    MEM: charOut charCpy isLT26 temp doesDec
            X       X     ^ 0     0     0
    >(temp)+++++[<+++++>-]<+            isLT26 = 26 (LT: less than)
    <(charCpy)[
        >(isLT26)[>(temp)+>(doesDec)[-]+<<-]
        >(temp)[<+>-]                   上一行和此行：isLT26非0时，使doesDec为1
        >(doesDec)[<<->>-]              若doesDec，则isLT26减1、doesDec减1
        <<<(charCopy)-                  charCopy减1
    ]

    如果是小写字母，输出对应大写字母；否则输出本身。
    MEM: charOut counter isLT26
            X     ^ 0       B
    >(isLT26)[                          isLT26非0时
        <(counter)++++[<++++++++>-]     charOut加32 (4 * 8)
        >(isLT26)[-]                    将isLT26清零，以退出循环
    ]
    <(counter)++++++++[<++++++++>-]<+   charOut加65 (8 * 8 加 1)
    (charOut).[-]                       输出并清零

    MEM: zero zero charIn
         ^ 0    0     0
    >>(charIn),+                        读取下一个字符
]
```

去掉注释是这样，151字节：

```brainfuck
>>,+[->++++++++[<-------->-]<-[<+<+>>-]>+++++[<+++++>-]<+<[>[>+>[-]+<<-]>[<+>-]>[<<->>-]<<<-]>[<++++[<++++++++>-]>[-]]<++++++++[<++++++++>-]<+.[-]>>,+]
```

然后天就亮了。

## 参考资料

[^SILI]: [project-epb/Chatbot-SILI ｜ GitHub](https://github.com/project-epb/Chatbot-SILI)

[^Turing]: [什么是图灵完备？ - Ran C的回答 ｜ 知乎](https://www.zhihu.com/question/20115374/answer/288346717)

[^enwp]: [Brainfuck ｜ Wikipedia](https://en.wikipedia.org/wiki/Brainfuck)
