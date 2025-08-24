TOSZ =24
ADDR = 2^(64 - 24) = 2^40

40 - 11 = 29	//除去 4K page， 剩余bit

29 = 2 + 9 + 9 + 9	//每级Table最大9bit,29bit划分为4级页表
[40 : 39][38:30][29:21][20:12][11:0]

L0 + L1 + L2 + L3(page)

block 有 
Level 0: 512 GB
Level 1: 1 GB
Level 2: 2 MB