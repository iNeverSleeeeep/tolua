
```c
// lj_func.c 这个是构造一个新的lua function的入口 lj_mem_newgco是构造GC相关 对于proto和evn的引用是通过set*ref的方式进行
static GCfunc *func_newL(lua_State *L, GCproto *pt, GCtab *env)
{
  uint32_t count;
  GCfunc *fn = (GCfunc *)lj_mem_newgco(L, sizeLfunc((MSize)pt->sizeuv));
  fn->l.gct = ~LJ_TFUNC;
  fn->l.ffid = FF_LUA;
  fn->l.nupvalues = 0;  /* Set to zero until upvalues are initialized. */
  /* NOBARRIER: Really a setgcref. But the GCfunc is new (marked white). */
  setmref(fn->l.pc, proto_bc(pt));
  setgcref(fn->l.env, obj2gco(env));
  /* Saturating 3 bit counter (0..7) for created closures. */
  count = (uint32_t)pt->flags + PROTO_CLCOUNT;
  pt->flags = (uint8_t)(count - ((count >> PROTO_CLC_BITS) & PROTO_CLCOUNT));
  return fn;
}
// lj_gc.c 构造gc相关 可以看到通过 setgcrefr 和 setgcref两个函数，把当前对象加入到了global_State的gc列表里面
void * LJ_FASTCALL lj_mem_newgco(lua_State *L, GCSize size)
{
  global_State *g = G(L);
  GCobj *o = (GCobj *)g->allocf(g->allocd, NULL, 0, size);
  if (o == NULL)
    lj_err_mem(L);
  lua_assert(checkptrGC(o));
  g->gc.total += size;
  setgcrefr(o->gch.nextgc, g->gc.root);
  setgcref(g->gc.root, o);
  newwhite(g, o);
  return o;
}

```
通过以上代码，可以看出，在构造新的lua function的时候，对于function和proto的绑定只有一个弱引用，并没有gc相关的引用
我们再继续看执行gc的时候是如何关联function和proto的
```c
// lj_gc.c 关注以下函数的执行
void lj_gc_fullgc(lua_State *L);
static size_t gc_onestep(lua_State *L);
static size_t propagatemark(global_State *g);

// 经过上面几个函数 对于function的标记是执行下面这个函数，我们开始仔细看
static void gc_traverse_func(global_State *g, GCfunc *fn)
{
  gc_markobj(g, tabref(fn->c.env)); // 这里标记了env
  if (isluafunc(fn)) {
    uint32_t i;
    lua_assert(fn->l.nupvalues <= funcproto(fn)->sizeuv);
    gc_markobj(g, funcproto(fn)); // 这里标记了proto
    for (i = 0; i < fn->l.nupvalues; i++)  // 标记了 upvalues
      gc_markobj(g, &gcref(fn->l.uvptr[i])->uv);
  } else {
    uint32_t i;
    for (i = 0; i < fn->c.nupvalues; i++)  /* Mark C function upvalues. */
      gc_marktv(g, &fn->c.upvalue[i]);
  }
}
```
通过上面的代码，我们可以确定，function gc的过程是动态查找关联proto和env的过程，而在gc链条里而没有被调用mark的gcobj，会被最终清除。
根据以上，推断运行时替换function的proto不会产生内存泄漏问题。
