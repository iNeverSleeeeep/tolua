
```c
// lfunc.c lua里面构造lua function的函数入口(在lua源码里面就是closure) 可以看到这个里面没有传递proto指针
// 而是通过luaC_link将closure添加到全局的gc列表中
Closure *luaF_newLclosure (lua_State *L, int nelems, Table *e) {
  Closure *c = cast(Closure *, luaM_malloc(L, sizeLclosure(nelems)));
  luaC_link(L, obj2gco(c), LUA_TFUNCTION);
  c->l.isC = 0;
  c->l.env = e;
  c->l.nupvalues = cast_byte(nelems);
  while (nelems--) c->l.upvals[nelems] = NULL;
  return c;
}
// lvm.c 看一下lua虚拟机解析lua源码构造function的地方
void luaV_execute (lua_State *L, int nexeccalls) {
  // ... 之前的逻辑
      case OP_CLOSURE: {
        Proto *p;
        Closure *ncl;
        int nup, j;
        p = cl->p->p[GETARG_Bx(i)];
        nup = p->nups;
        ncl = luaF_newLclosure(L, nup, cl->env); // 构造了function
        ncl->l.p = p; // 设置了proto
        for (j=0; j<nup; j++, pc++) {
          if (GET_OPCODE(*pc) == OP_GETUPVAL)
            ncl->l.upvals[j] = cl->upvals[GETARG_B(*pc)];
          else {
            lua_assert(GET_OPCODE(*pc) == OP_MOVE);
            ncl->l.upvals[j] = luaF_findupval(L, base + GETARG_B(*pc));
          }
        }
        setclvalue(L, ra, ncl);
        Protect(luaC_checkGC(L));
        continue;
      }
  // ... 之后的逻辑
}
```
通过以上代码，我们了解了lua在构造function的时候，function和proto没有gc上的关联，只是存在了一个引用
下面我们看一下进行gc的时候，如果将function和proto关联起来的。
```c
// lgc.c 关注以下函数顺序执行
void luaC_fullgc (lua_State *L);
static l_mem singlestep (lua_State *L);
static l_mem propagatemark (global_State *g);

// 这里进入了function的标记流程
static void traverseclosure (global_State *g, Closure *cl) {
  markobject(g, cl->c.env);
  if (cl->c.isC) {
    int i;
    for (i=0; i<cl->c.nupvalues; i++)  /* mark its upvalues */
      markvalue(g, &cl->c.upvalue[i]);
  }
  else {
    int i;
    lua_assert(cl->l.nupvalues == cl->l.p->nups);
    markobject(g, cl->l.p); // 这里标记了proto
    for (i=0; i<cl->l.nupvalues; i++)  /* mark its upvalues */
      markobject(g, cl->l.upvals[i]);
  }
}
```
通过以上我们了解到，lua function是在gc过程中动态查找的proto指针执行标记，在global_State的gc列表中而没有被标记的gcobj最终会被清除。
得出结论，我们通过代码替换函数的proto指针，不会产生gc问题。
