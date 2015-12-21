# bytework
A replacement for [`byteplay`][1] for Python 3.4+

NOTE: THIS IS JUST A PLACEHOLDER. BYTEWORK IS NOT IMPLEMENTED YET!

Noam Yorav-Raphael's  `byteplay` module is the coolest thing since sliced `co_code`, but it has one major problem: It doesn't work with Python 3. I found four incomplete ports to Python 3, and [tried it myself][2], but after a bit of hacking I realized that it wouldn't be a simple fix, and it would be easier to just rewrite things from scratch. (There are other alternatives, like the nifty assembler [`maynard`][4], but they don't do quite the same thing.) Once you're going to rewrite things from scratch, you might as well use all the nifty features added in Python 3.4. For example, the [`dis`][3] module now contains actual `Bytecode` and `Instruction` objects that do a lot of the work for us.

The point of `bytework`, like `byteplay` and `bytecodehacks`, is to be able to hack up some bytecode in simple ways, knowing you're not going to break anything besides what you wanted to break.

A `bytework.Code` object wraps a builtin `code` object, and provides an interface that's like `code` and `dis.Bytecode` in one, and as mutable as possible. For example, it has a `co_names`, just like the underlying `code`, but you can assign to it. And it's can iterate `Instruction` objects, just like a `Bytecode` object, but it's also a mutable sequence, not just an iterable. And its `Instruction` objects are easier to construct (e.g., with `dis`,you might have to write `Instruction('LOAD_CONST', opmap['LOAD_CONST'], 2, c.co_consts[2], repr(c.co_consts[2]), old_instr.offset, old_instr.starts_line, False)`, while with `bytework` it's just `Instruction(LOAD_CONST, arg=2)`.

Here's a complete example of a decorator that turns all of a function's global and builtin lookups into constants:

    def optimize_globals(f):
        code = bytework.Code(f)
        globals = f.__globals__
        builtins = globals.get('__builtin__', {})
        if isinstance(builtins, types.ModuleType):
            builtins = builtins.__dict__
        globals = ChainMap(globals, builtins)
        cache = {}
        for i, name in enumerate(code.co_names):
            try:
                value = globals[name]
                cache[i] = len(code.co_consts)
                code.co_consts.append(value)
            except KeyError:
                pass
        for i, instr in enumerate(code):
            if instr.opcode == bytework.LOAD_GLOBAL and instr.arg in cache:
                code[i] = Instruction(bytework.LOAD_CONST, arg=cache[instr.arg])
        f.__code__ = code.to_code()
        return f

    [1] https://wiki.python.org/moin/ByteplayDoc
    [2] https://github.com/abarnert/byteplay
    [3] https://docs.python.org/3/library/dis.html
    [4] https://bitbucket.org/larry/maynard
