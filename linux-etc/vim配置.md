## vim配置



### 1. 下载插件



www.vim.org



Scripts

Browse all



分别下载插件：

Source Exporer

NERD Tree

Tag List



```
# mkdir .vim
# cd .vim
# mkdir plugin
# ls
SrcExpl-master.zip
nerdtree-master.zip
taglist_46.zip
plugin
```



### 2. 解压

```
# unzip SrcExpl-master.zip
# unzip nerdtree-master.zip
# unzip taglist_46.zip
```



### 3. 加载插件

```
# cp  SrcExpl-master/plugin/ plugin/ -rf
# cp nerdtree-master/plugin/NERD_tree.vim plugin/ -rf
```





### 4. vim环境设置

```
# vim ~/.vimrc
  1 set nu
  2 set ff=unix
  3 set hlsearch
  4 set incsearch
  5 set helplang=cn
  6 set encoding=utf-8
  7 set tabstop=4
  8 set shiftwidth=4
  9 set autoindent
 10 set cindent
 11 
 12 set tags+=~/tool/linux-4.14.172/tags
 13 set tags+=~/tool/u-boot-2020.01/tags
 14 
 15 
 16 "===================================
 17 "Tag List set
 18 filetype on
 19 nmap <F9> :TlistToggle<CR>
 20 let Tlist_Ctags_Cmd = "/usr/bin/ctags"
 21 let Tlist_Inc_WinWidth = 0
 22 let Tlist_Exit_OnlyWindow = 0
 23 
 24 let Tlist_Auto_Open = 0
 25 let Tlist_Use_Right_Window = 1
 26 
 27 "===================================
 28 "Source Explorer set
 29 nmap <F8> :SrcExplToggle<CR>
 30 nmap <C-H> <C-W>h   "Move to left window
 31 nmap <C-J> <C-W>j   "Move to down window
 32 nmap <C-K> <C-W>k   "Move to up window
 33 nmap <C-L> <C-W>l   "Move to right window
 34 
 35 let g:SrcExpl_winHeight = 8
 36 let g:SrcExpl_refreshTime = 100
 37 let g:SrcExpl_jumpKey = "<ENTER>"
 38 let g:SrcExpl_gobackKey = "<SPACE>"
 39 let g:SrcExpl_isUpdateTags = 0
 40 
 41 
 42 "===================================
 43 "NERD Tree set
 44 let NERDTreeWinPos = "left"
 45 nmap <F7> :NERDTreeToggle<CR>
 46 
```









