a.txt

```
# Vulnerable
/zzz(a|ab|b)*kk/
```

`RegexScanner.ml` 第35行，`let p = ParsingMain.parse_pattern lexbuf` 的输出值为

```
p: ParsingData.pattern =
  (r, 8)

r: ParsingData.regex = 
  (Conc
    ((Atom (Char 'z'),
      {spos = 0; epos = 0; scount = 0; nullable = false; cflags = 0}),
    (Conc
      ((Atom (Char 'z'),
        {spos = 1; epos = 1; scount = 0; nullable = false; cflags = 0}),
      (Conc
        ((Atom (Char 'z'),
          {spos = 2; epos = 2; scount = 0; nullable = false; cflags = 0}),
        (Conc
          ((Kleene (Gq,
             (Group (CAP 1, 0, 0,
               (Alt
                 ((Atom (Char 'a'),
                   {spos = 4; epos = 4; scount = 0; nullable = false;
                    cflags = 0}),
                 (Alt
                   ((Conc
                      ((Atom (Char 'a'),
                         {spos = 6; epos = 6; scount = 0; nullable = false;
                          cflags = 0}),
                        (Atom (Char 'a'),
                          {spos = 7; epos = 7; scount = 0; nullable = false;
                           cflags = 0})),
                        {spos = 6; epos = 7; scount = 0; nullable = false;
                         cflags = 0}),
                      (Atom (Char 'b'),
                       {spos = 9; epos = 9; scount = 0; nullable = false;
                        cflags = 0})),
                     {spos = 6; epos = 9; scount = 0; nullable = false;
                      cflags = 0})),
                  {spos = 4; epos = 9; scount = 0; nullable = false;
                   cflags = 0})),
                {spos = 3; epos = 10; scount = 0; nullable = false;
                 cflags = 0})),
              {spos = 3; epos = 11; scount = 0; nullable = false; cflags = 0}),
             (Conc
               ((Atom (Char 'k'),
                 {spos = 12; epos = 12; scount = 0; nullable = false;
                  cflags = 0}),
               (Atom (Char 'k'),
                {spos = 13; epos = 13; scount = 0; nullable = false;
                 cflags = 0})),
              {spos = 12; epos = 13; scount = 0; nullable = false;
               cflags = 0})),
            {spos = 3; epos = 13; scount = 0; nullable = false; cflags = 0})),
         {spos = 2; epos = 13; scount = 0; nullable = false; cflags = 0})),
       {spos = 1; epos = 13; scount = 0; nullable = false; cflags = 0})),
     {spos = 0; epos = 15; scount = 0; nullable = false; cflags = 0})
     
结构图：

  // zzz(a|ab|b)*kk

  Conc
    Atom (Char 'z')
    Conc
      Atom (Char 'z')
      Conc
        Atom (Char 'z')
        Conc
          Kleene Gq
             Group
               CAP
               1
               0
               0
               Alt
                 Atom (Char 'a')
                 Alt
                   Conc
                     Atom (Char 'a')
                     Atom (Char 'b')
                   Atom (Char 'b')
        Conc
          Atom (Char 'k'),
          Atom (Char 'k'),

```

一个巨大的tuple，通过这个tuple的结构，可以得知`ParsingData.pattern`，即Regex语法解析的各个字段的含义。


转换后的NFA：

```
nfa: t =
  {states =
    [|Match ([('z', 'z')], 1); Match ([('z', 'z')], 2);
      Match ([('z', 'z')], 11); BeginCap (1, 4); BranchAlt (5, 6);
      Match ([('a', 'a')], 10); BranchAlt (7, 9); Match ([('a', 'a')], 8);
      Match ([('b', 'b')], 10); Match ([('b', 'b')], 10); EndCap (1, 11);
      BranchKln (true, 3, 12); Match ([('k', 'k')], 13);
      Match ([('k', 'k')], 14); End|];
   transitions =
    [|
    Some [('z', 'z', 1)];
    Some [('z', 'z', 2)];
    Some [('z', 'z', 11)];
    None; = @10
    None; = @10
    Some [('a', 'a', 10)];
    Some [('a', 'a', 8); ('b', 'b', 10)];
    Some [('a', 'a', 8)];
    Some [('b', 'b', 10)];
    Some [('b', 'b', 10)];
    Some [('k', 'k', 13); ('a', 'a', 10); ('a', 'a', 8); ('b', 'b', 10)];
    None; = @10
    Some [('k', 'k', 13)];
    Some [('k', 'k', 14)];
    Some [(zmin, zmax, 14)]; = @End
    |];
   positions =
    [|(0, 0); (1, 1); (2, 2); (3, 3); (4, 9); (4, 4); (6, 9); (6, 6);
      (7, 7); (9, 9); (10, 10); (3, 11); (12, 12); (13, 13); (15, 15)|];
   root = 0}

```

NFA的立体结构

```
0('z') -> 1('z') -> 2('z') -> 11('*') -> 12('k') -> 13('k') -> 14(End)
                             /    ^
                            /      \
                           V        \
                       3('(')        \
                          V           \
                      4('|')           \
                         / \            \
                        V   V            \ 
                     5('a') 6('|')        \
                       |      / \         |
                       |     V   V        |
                       | 7('a')  9('b')   |
                       |    V      |     /
                       | 8('b')    |    /
                        \    |    /    /
                         V   V   V    /
                           10(')') __/
```

`nfa.t.transitions` 表示的是消去 $\epsilon$ 的NFA转移路径。






