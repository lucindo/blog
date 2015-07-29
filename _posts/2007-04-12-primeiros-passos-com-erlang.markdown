---
layout: post
title:  "Primeiros passos com Erlang"
date:   2007-04-12 20:59:12
categories: programming erlang
comments: true
---

Nada melhor do que programar para aprender uma linguagem de programação. Por isso comecei um pequeno projeto em [Erlang][erlang-link]. Faz parte desse projeto um cliente do protocolo [NNTP][nntp-link]. Implementei apenas os comandos NNTP que eu precisava.

Essa implementação está bem crua ainda, sem tratamento de erros do protocolo, que se acontecerem vão resultar em _crash_ por _badmatch_.

{% highlight erlang %}
-module(nntp).
-vsn(1).
-author({renato, lucindo}).
-purpose(”work :-P “).

%% Implements a little client for:
%%  NNTP: http://tools.ietf.org/html/rfc977
%%  Simple NNTP Auth: http://tools.ietf.org/html/rfc4643

%% protocol functions
-export([connect/2, connect/4, cmd_quit/1]).
-export([cmd_list/1, cmd_group/2, cmd_stat/2, cmd_head/2, cmd_body/2]).
%% utils
-export([get_header_contents/2]).

-include(”nntp.hrl“).

%% Little documentation:
%% warning 1:
%%   You must include “nntp.hrl” to the definition of newsgroup record.
%%
%% Warning 2:
%%   This is a quick and really dirty implementation. Will result on a badmath
%%   error due protocol erros. This will be fixed one day (maybe with wappers
%%   functions with try…catch).
%%
%% Commands (functions exported):
%%
%% nntp:connect(Host, Port)
%%   Simple connection on Host:Port
%%   Returns {ok, Socket} on success. {error, Reason} otherwise
%%
%% nntp:connect(Host, Port, User, Password)
%%   Connect and authenticate on Host:Port. Same return values as connect/2
%%
%% nntp:cmd_quit(Sockt)
%%   Terminate de NNTP session and close the socket.
%%   Aways returns ok.
%%
%% nntp:cmd_list(Socket)
%%   List all groups on server.
%%   Resturns a erlang:list of newsgroups record (see “nntp.hrl”).
%%
%% nntp:cmd_group(Socket, GroupName)
%%   Selects the group GroupName.
%%   Returns a tuple {ok, NMessages, First, Last}, where NMessages is the number of
%%   messages on group, First is the first article number and Last is the last
%%   article number
%%
%% nntp:cmd_stat(Socket, Msgnum)
%%   Selects the message Msgnum (message number) for reading
%%   Returns {ok, MessageID} where MessageID is the message unique ID
%%
%% nntp:cmd_head(Socket, Msgnum)
%%   Gets the headers of article Msgnum.
%%   Returns a tuple {ok, MessageID, Headers} where Headers is a list of strings
%%
%% nntp:get_header_contents(Header, Field)
%%   Gets the contents of a Header’s Field (the header as returned by nntp:cmd_head)
%%   Returns {ok, Filed, Content} on success, error otherwise
%%
%% nntp:cmd_body(Socket, Msgnum)
%%   Gets the body of message Msgnum as string.
%%   Returns {ok, MessageID, Message} on success, error otherwise

connect(Hostname, Port) ->
    case gen_tcp:connect(Hostname, Port, [binary, {packet, line}, {active, false}]) of
        {ok, Socket} ->
            case read_response(Socket) of
                {ok, Code, _} when Code == 200; Code == 201 ->
                    {ok, Socket};
                _ ->
                    gen_tcp:close(Socket),
                    {error, “Greetings read error“}
            end;
        _ ->
            {error, “Connection error“}
    end.
connect(Hostname, Port, User, Pass) ->
    {ok, Socket} = connect(Hostname, Port),
    auth(Socket, User, Pass).

auth(Socket, User, Pass) ->
    ok = send_request(Socket, “AUTHINFO USER ” ++ User),
    {ok, 381, _} = read_response(Socket),
    ok = send_request(Socket, “AUTHINFO PASS ” ++ Pass),
    {ok, 281, _} = read_response(Socket),
    {ok, Socket}.

cmd_quit(Socket) ->
    ok = send_request(Socket, “QUIT“),
    {ok, 205, _} = read_response(Socket),
    ok = gen_tcp:close(Socket),
    ok.

cmd_list(Socket) ->
    ok = send_request(Socket, “LIST“),
    {ok, 215, _} = read_response(Socket),
    cmd_list(Socket, []).
cmd_list(Socket, Glist) ->
    case gen_tcp:recv(Socket, 0) of
        {ok, Response} ->
            Line = remove_rn(binary_to_list(Response)),
            if
                Line == “.” ->
                    Glist;
                true ->
                    Tk = string:tokens(Line, “ “),
                    Name = lists:nth(1, Tk),
                    Last = list_to_integer(lists:nth(2, Tk)),
                    First = list_to_integer(lists:nth(3, Tk)),
                    CanPost = list_to_atom(lists:nth(4, Tk)),
                    Group = #newsgroup {
                      name = Name,
                      last = Last,
                      first = First,
                      canpost = CanPost
                     },
                    cmd_list(Socket, lists:append(Glist, [Group]))
            end;
        _ ->
            error
    end.

cmd_group(Socket, Group) ->
    ok = send_request(Socket, “GROUP ” ++ Group),
    {ok, 211, GrpInfo} = read_response(Socket),
    Tk = string:tokens(GrpInfo, “ “),
    NMessages = list_to_integer(lists:nth(1, Tk)),
    First = list_to_integer(lists:nth(2, Tk)),
    Last = list_to_integer(lists:nth(3, Tk)),
    {ok, NMessages, First, Last}.

cmd_stat(Socket, Msgnum) ->
    ok = send_request(Socket, “STAT “++ integer_to_list(Msgnum)),
    {ok, 223, Response} = read_response(Socket),
    {ok, lists:nth(2, string:tokens(Response, “ “))}.

cmd_head(Socket, Msgnum) ->
    exec_cmd_response_and_stream(Socket, Msgnum, “HEAD “, 221, fun read_lines_as_list/3).

cmd_body(Socket, Msgnum) ->
    exec_cmd_response_and_stream(Socket, Msgnum, “BODY “, 222, fun read_lines_as_string/3).

exec_cmd_response_and_stream(Socket, Msgnum, Command, RespCode, ReadFunction) when integer(Msgnum) ->
    exec_cmd_response_and_stream(Socket, integer_to_list(Msgnum), Command, RespCode, ReadFunction);
exec_cmd_response_and_stream(Socket, Msgnum, Command, RespCode, ReadFunction) ->
    ok = send_request(Socket, Command ++ Msgnum),
    {ok, RespCode, Response} = read_response(Socket),
    {ok, lists:nth(2, string:tokens(Response, “ “)), apply(ReadFunction, [Socket, “.“, []])}.

read_lines_as_list(Socket, Delim, Str) ->
    case gen_tcp:recv(Socket, 0) of
        {ok, Response} ->
            Line = remove_rn(binary_to_list(Response)),
            if
                Line == Delim ->
                    Str;
                true ->
                    read_lines_as_list(Socket, Delim, lists:append(Str, [Line]))
            end;
        _ ->
            error
    end.

read_lines_as_string(Socket, Delim, Str) ->
    case gen_tcp:recv(Socket, 0) of
        {ok, Response} ->
            StrR = binary_to_list(Response),
            Line = remove_rn(StrR),
            if
                Line == Delim ->
                    Str;
                true ->
                    read_lines_as_string(Socket, Delim, [Str, StrR])
            end;
        _ ->
            error
    end.

send_request(Socket, Cmd) ->
    Str = Cmd ++ “rn“,
    case gen_tcp:send(Socket, Str) of
        ok->
            ok;
        {error, Reason} ->
            io:format(”error on send socket: ~w~n“, [Reason]),
            gen_tcp:close(Socket),
            error
    end.

read_response(Socket) ->
    case gen_tcp:recv(Socket, 0) of
        {ok, Response} ->
            parse_response(Response);
        {error, Reason} ->
            io:format(”error on recv socket: ~w~n“, [Reason]),
            gen_tcp:close(Socket),
            error
    end.

parse_response(Response) ->
    <<Code:3/binary, _:1/binary, Message/binary>> = Response,
    {ok, list_to_integer(binary_to_list(Code)), remove_rn(binary_to_list(Message))}.

get_header_contents(Header, Field) ->
    get_header_contents_by_field(Field, Header).

get_header_contents_by_field(Field, [H|T]) ->
    Tk = string:tokens(H, “: “),
    if
        length(Tk) > 1 ->
            F = lists:nth(1, Tk),
            C = lists:nth(2, Tk),
            if
                F == Field ->
                    {ok, F, C};
                true ->
                    get_header_contents_by_field(Field, T)
            end;
        true ->
            get_header_contents_by_field(Field, T)
    end;
get_header_contents_by_field(_, []) ->
    error.

remove_rn(Str) ->
    string:strip(string:strip(Str, both, $n), both, $r).
{% endhighlight %}

O arquivo nntp.hrl tem apenas:

{% highlight erlang %}
-record(newsgroup,
        {
          name = “”,
          last = 0,
          first = 0,
          canpost = n
         }).
{% endhighlight %}

Alguém sabe como exportar records de uma maneira fácil?

[erlang-link]: http://www.erlang.org
[nntp-link]: https://en.wikipedia.org/wiki/Network_News_Transfer_Protocol
