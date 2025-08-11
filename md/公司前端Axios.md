# 前端Axios

##### post请求

​	请求前弹窗是否确认，确认后发送请求。

```js
batchRegister(selections) {
      const args = _.map(selections, (vo) => _.pick(vo, ['id', 'deviceId']))
      this.msg = []
      console.log(selections, 'selections')
      this.$confirm(`您确认要批量注册吗?`, '温馨提示', {
        confirmButtonText: '立即执行',
        cancelButtonText: '考虑一下',
        type: 'warning'
      }).then(() => {
        this.$post('url', args)
          .then(() => {
            this.$notify({
              title: '温馨提示',
              message: '注册成功',
              type: 'success'
            })
            this.dialogVisible = false
            this.crud.refresh()
          })
          .catch((error) => {
            this.dialogVisible = true
            this.msg = error.split('||')
            this.crud.refresh()
          })
      })
    },
```

