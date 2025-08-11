#### 配置路由

path:"/search/:word" 动态路由

```vue
const routes = [
{
        path: "/search/:word",
        name: "search",
        component: search,
        props: true
    }
]
```

路由调用者

```vue
function toSearch(word:any){
  router.push({name: 'search', params: {word}})
}
```

路由目的地

```vue
const route = useRoute()
let word = route.params.word
```

