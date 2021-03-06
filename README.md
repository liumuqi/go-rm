# Go Redis module
go-rm will let you write redis module in golang.

Read in | [中文](./README-zh_CN.md) | [English](./README.md) | [Spanish](./README-es.md)

## Modules

* [redismodule](https://github.com/redismodule)
    * [rxhash](https://github.com/redismodule/rxhash)

## Demo

```bash
# Ensure you installed the newest redis
# for example by using brew you can
# brew reinstall redis --HEAD

# Build redis module
go build -v -buildmode=c-shared github.com/redismodule/rxhash/cmd/rxhash

# Start redis-server and load our module with debug log
redis-server --loadmodule rxhash --loglevel debug
```

__Connect to out redis-server__

```bash
# Test hgetset
redis-cli hset a a 1
#> (integer) 1
redis-cli hgetset a a 2
#> "1"
redis-cli hget a a
#> "2"
# Return nil if field not exists
redis-cli hgetset a b 2
#> (nil)
redis-cli hgetset a b 3
#> "2"
```

Wow, it works, now you can distribute this redis module to you friends. :P

## How to write a module

Implement a redis module is as easy as you write a cli app in go, this is all you need to implement above command.

```go
package main

import "github.com/wenerme/go-rm/rm"

func main() {
    // In case someone try to run this
    rm.Run()
}

func init() {
    rm.Mod = CreateMyMod()
}
func CreateMyMod() *rm.Module {
    mod := rm.NewMod()
    mod.Name = "hashex"
    mod.Version = 1
    mod.Commands = []rm.Command{CreateCommand_HGETSET()}
    return mod
}
func CreateCommand_HGETSET() rm.Command {
	return rm.Command{
		Usage: "HGETSET key field value",
		Desc: `Sets the 'field' in Hash 'key' to 'value' and returns the previous value, if any.
Reply: String, the previous value or NULL if 'field' didn't exist. `,
		Name:   "hgetset",
		Flags:  "write fast deny-oom",
		FirstKey:1, LastKey:1, KeyStep:1,
		Action: func(cmd rm.CmdContext) int {
			ctx, args := cmd.Ctx, cmd.Args
			if len(cmd.Args) != 4 {
				return ctx.WrongArity()
			}
			ctx.AutoMemory()
			key, ok := openHashKey(ctx, args[1])
			if !ok {
				return rm.ERR
			}
			// get the current value of the hash element
			var val rm.String;
			key.HashGet(rm.HASH_NONE, cmd.Args[2], (*uintptr)(&val))
			// set the element to the new value
			key.HashSet(rm.HASH_NONE, cmd.Args[2], cmd.Args[3])
			if val.IsNull() {
				ctx.ReplyWithNull()
			} else {
				ctx.ReplyWithString(val)
			}
			return rm.OK
		},
	}
}
// open the key and make sure it is indeed a Hash and not empty
func openHashKey(ctx rm.Ctx, k rm.String) (rm.Key, bool) {
	key := ctx.OpenKey(k, rm.READ | rm.WRITE)
	if key.KeyType() != rm.KEYTYPE_EMPTY && key.KeyType() != rm.KEYTYPE_HASH {
		ctx.ReplyWithError(rm.ERRORMSG_WRONGTYPE)
		return rm.Key(0), false
	}
	return key, true
}
```

## Fantasy

* A module management module, supplies
    * mod.search
        * Search module from repository(github?)
        * Repository structure like this
        ```
        /namespace
            /module-name
                /bin
                    /darwin_amd64
                        module-name.so
                        module-name.sha
                    /linux_amd64
                module-name.go     
        ```
    * mod.get
        * Download module to ~/.redismodule
        * Because module is write in go, so we can build for almost any platform
        * We can use tag/commit to version the binary, so we can download the old version too
    * mod.install
        * Install downloaded module by calling redis command
    * ...
* A cluster management module
    * Easy to create/manage/monitor redis3 cluster
* A json data type to demonstration how to add new data type in redis.
    * json.fmt key template
    * json.path key path \[pretty]
    * json.get key \[pretty]
    * json.set key value
        * this will validate the json format

## Pitfall
* C can not call Go function, so every callback is pre-generated
    * 200 commands at most
    * 5 data type at most
    * limits are easy to change, just need a proper max value
* Go can not call var_args, function call is pre-generated
    * HashSet/HashGet can accept 20 args at most
    * limits are easy to change, just need a proper max value
* Don't know what happens when unload a golang shared module
    * Single module
    * Multi module
        * Is there runtime are shared ?
* Module write in go can not report it's memory usage to redis, max memory limits is useless
* If a module write in go also include a third party write in other language, the memory usage is unknown
* Module can only accept command, seems there is no way to call redis initiative.

## TODO

* Find a proper limits for data types and var_args
