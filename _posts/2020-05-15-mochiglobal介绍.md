# `mochiglobal`介绍

对于2000qps的并发处理过程中，即使是设置了并发读写的ets表对于1K以上的数据的读取也存在明显的性能瓶颈。

```
ets:new(person, [set, public, named_table, **{write_concurrency, true}, {read_concurrency, true}** ]).
```

为此，我们使用了mochiweb工程中的一个组件，`mochiglobal`。

```
https://github.com/mochi/mochiweb.git
```

`mochiglobal`的源码如下

```
%% @author Bob Ippolito <bob@mochimedia.com>
%% @copyright 2010 Mochi Media, Inc.
%% @doc Abuse module constant pools as a "read-only shared heap" (since erts 5.6)
%%      <a href="http://www.erlang.org/pipermail/erlang-questions/2009-March/042503.html">[1]</a>.
-module(mochiglobal).
-author("Bob Ippolito <bob@mochimedia.com>").
-export([get/1, get/2, put/2, delete/1]).

-spec get(atom()) -> any() | undefined.
%% @equiv get(K, undefined)
get(K) ->
    get(K, undefined).

-spec get(atom(), T) -> any() | T.
%% @doc Get the term for K or return Default.
get(K, Default) ->
    get(K, Default, key_to_module(K)).

get(_K, Default, Mod) ->
    try Mod:term()
    catch error:undef ->
            Default
    end.

-spec put(atom(), any()) -> ok.
%% @doc Store term V at K, replaces an existing term if present.
put(K, V) ->
    put(K, V, key_to_module(K)).

put(_K, V, Mod) ->
    Bin = compile(Mod, V),
    code:purge(Mod),
    {module, Mod} = code:load_binary(Mod, atom_to_list(Mod) ++ ".erl", Bin),
    ok.

-spec delete(atom()) -> boolean().
%% @doc Delete term stored at K, no-op if non-existent.
delete(K) ->
    delete(K, key_to_module(K)).

delete(_K, Mod) ->
    code:purge(Mod),
    code:delete(Mod).

-spec key_to_module(atom()) -> atom().
key_to_module(K) ->
    list_to_atom("mochiglobal:" ++ atom_to_list(K)).

-spec compile(atom(), any()) -> binary().
compile(Module, T) ->
    {ok, Module, Bin} = compile:forms(forms(Module, T),
                                      [verbose, report_errors]),
    Bin.

-spec forms(atom(), any()) -> [erl_syntax:syntaxTree()].
forms(Module, T) ->
    [erl_syntax:revert(X) || X <- term_to_abstract(Module, term, T)].

-spec term_to_abstract(atom(), atom(), any()) -> [erl_syntax:syntaxTree()].
term_to_abstract(Module, Getter, T) ->
    [%% -module(Module).
     erl_syntax:attribute(
       erl_syntax:atom(module),
       [erl_syntax:atom(Module)]),
     %% -export([Getter/0]).
     erl_syntax:attribute(
       erl_syntax:atom(export),
       [erl_syntax:list(
         [erl_syntax:arity_qualifier(
            erl_syntax:atom(Getter),
            erl_syntax:integer(0))])]),
     %% Getter() -> T.
     erl_syntax:function(
       erl_syntax:atom(Getter),
       [erl_syntax:clause([], none, [erl_syntax:abstract(T)])])].
```



代码的使用比较简单，就是简单的put(K,V)存储数据，get(K,V)读取数据。

实际测试表明，代码的读性能和写死到代码里差不多，写性能就不用测了，肯定很差。所以这个模块对于那种多读少写的应用场景还是很好用的。

代码的原理是通过动态编译将Value存到一个模块的函数内，使用get时调用该函数即可。