# 登录逻辑

## 文件目录

这里的逻辑设计了这么几个文件，分别是

```markdown
- `page.vue` (src\views\system\login\page.vue)
- `base.vue` (src\views\system\login\base.vue)  基础组件  包含了一些通用的方法、数据和生命周期钩子等，用于在其他组件中复用
- `api.js` (src\views\system\login\api.js) 基础api文件存储地方
- `account.js` (src\store\modules\d2admin\modules\account.js) Vuex Store action对于account模块的定义的地方
- `util.cookies.js` (src\libs\util.cookies.js) cookie处理工具文件
- `util.js` (src\libs\util.js) 基础工具处理文件
```

## 代码逻辑解析

1. 用户在page.vue登录页面输入用户名、密码和验证码。

```html
<el-form-item
            prop="captcha"
            v-if="captchaState"
            :rules="{required: true,message: '请输入验证码',trigger: 'blur'}"
          >
            <el-input
              type="text"
              v-model="formLogin.captcha"
              placeholder="验证码"
              @keyup.enter.native="submit"
            >
              <template slot="append">
                <img
                  alt="请检查后端是否正常~~"
                  class="login-code"
                  style="cursor: pointer;width:145px;height: 33px;"
                  height="33px"
                  width="145px"
                  slot="suffix"
                  :src="image_base"
                  @click="getCaptcha"
                />
              </template>
            </el-input>
          </el-form-item>
        </el-form>
        <el-row v-if="isTenant && isPublic">
          <el-col :span="11">
            <button class="btn btn-primary btn-block" style="padding: 10px 10px;" @click="submit">登录</button>
          </el-col>
          <el-col :span="11" :offset="2">
            <button
              class="btn btn-primary btn-block"
              style="padding: 10px 10px;background-color: #409eff;color: #fff;"
              @click="$router.push('/register')">
              免费试用
            </button>
          </el-col>
        </el-row>
        <button v-else class="btn btn-primary btn-block" style="padding: 10px 10px;" @click="submit">登录</button>
        <component v-if="componentTag" :is="componentTag"></component>
```

2. 用户点击登录按钮，触发`base.vue`组件的`submit`方法。这个方法首先通过`this.$refs.loginForm.validate`进行表单验证，如果数据有效，就调用`this.login`方法。

```html
<button v-else class="btn btn-primary btn-block" style="padding: 10px 10px;" @click="submit">登录</button>
```

```javascript
    /**
     * @description 提交表单
     */
    // 提交登录信息
    submit () {
      const that = this
      this.$refs.loginForm.validate((valid) => {
        if (valid) {
          // 登录
          // 注意 这里的演示没有传验证码
          // 具体需要传递的数据请自行修改代码
          this.login({
            username: that.formLogin.username,
            password: that.$md5(that.formLogin.password),
            captcha: that.formLogin.captcha,
            captchaKey: that.captchaKey
          })
            .then(() => {
              // 重定向对象不存在则返回顶层路径
              // this.$router.replace(this.$route.query.redirect || '/')
              this.$router.replace('/')
            })
            .catch(() => {
              this.getCaptcha()
            })
        } else {
          // 登录表单校验失败
          this.$message.error('表单校验失败，请检查')
        }
      })
    },
```

3. `this.login`方法是通过Vuex的辅助函数`mapActions`绑定的`d2admin/account`模块下的`login` action。在`...mapActions('d2admin/account', ['login'])`这行代码中，`mapActions`函数接收两个参数，第一个是命名空间`'d2admin/account'`，第二个是一个数组，数组中的元素是需要绑定的action的名称。这样，`login` action就可以在`base.vue`组件中通过`this.login`来调用。注意这里的login方法可以从d2admin/account.js中去找寻，是自定义的一个文件，用于存储Vuex的辅助函数而已

```javascript
// base.vue
methods: {
  ...mapActions('d2admin/account', ['login']),
  ...
}
```

4. `login` action在`d2admin/account.js`模块的Vuex store中定义，它接收一个对象作为参数，该对象包含用户名、密码和验证码等信息。

```javascript
// d2admin/account.js
actions: {
  async login ({ dispatch }, {
    username = '',
    password = '',
    captcha = '',
    captchaKey = ''
  } = {}) {
    ...
  },
  ...
}
```

5. account.js的`login` action内部首先调用了`SYS_USER_LOGIN`函数，这个函数使用`request`方法向服务器发送一个包含登录信息的请求。

```javascript
// d2admin/account.js
let res = await SYS_USER_LOGIN({
  username,
  password,
  captcha,
  captchaKey
})
```

`SYS_USER_LOGIN`这个方法定义在了`api.js`文件中，如下，主要是通过`get`请求向服务器请求数据

```javascript
import { request } from '@/api/service'

export function SYS_USER_LOGIN (data) {
  return request({
    url: 'api/login/',
    method: 'post',
    data
  })
}
```

这里的request是在service.js文件中定义的一个函数，可见这里的request先会对数据进行一次封装，最终返回一个service，service是这个js文件中的createService方法，用于创建一个axios请求，用于针对三个请求周期，请求拦截修饰、响应拦截修饰、错误消息修饰等，可以详细看下代码。

```javascript

function createService () {
  // 创建一个 axios 实例
  const service = axios.create({
    baseURL: util.baseURL(),
    timeout: 20000,
    paramsSerializer: (params) => qs.stringify(params, { indices: false })
  })
  // 请求拦截
  service.interceptors.request.use(
    config => config,
    error => {
      // 发送失败
      console.log(error)
      return Promise.reject(error)
    }
  )
  // 响应拦截
  service.interceptors.response.use(
    async response => {
      // dataAxios 是 axios 返回数据中的 data
      let dataAxios = response.data || null
      if (response.headers['content-disposition']) {
        dataAxios = response
      }
      // 这个状态码是和后端约定的
      const { code } = dataAxios
      // 根据 code 进行判断
      if (code === undefined) {
        // 如果没有 code 代表这不是项目后端开发的接口 比如可能是 D2Admin 请求最新版本
        return dataAxios
      } else {
        // 有 code 代表这是一个后端接口 可以进行进一步的判断
        switch (code) {
          case 2000:
            // [ 示例 ] code === 2000 代表没有错误
            // TODO 可能结果还需要code和msg进行后续处理，所以去掉.data返回全部结果
            // return dataAxios.data
            return dataAxios
          case 401:
            if (response.config.url === 'api/login/') {
              errorCreate(`${getErrorMessage(dataAxios.msg)}`)
              break
            }
            var res = await refreshTken()
            // 设置请求超时次数
            var config = response.config
            util.cookies.set('token', res.data.access)
            config.headers.Authorization = 'JWT ' + res.data.access
            config.__retryCount = config.__retryCount || 0
            if (config.__retryCount >= config.retry) {
              // 如果重试次数超过3次则跳转登录页面
              util.cookies.remove('token')
              util.cookies.remove('uuid')
              router.push({ path: '/login' })
              errorCreate('认证已失效,请重新登录~')
              break
            }
            config.__retryCount += 1
            return service(config)
          case 404:
            dataNotFound(`${dataAxios.msg}`)
            break
          case 4000:
            // 删除cookie
            errorCreate(`${getErrorMessage(dataAxios.msg)}`)
            break
          case 400:
            errorCreate(`${dataAxios.msg}`)
            break
          default:
            // 不是正确的 code
            errorCreate(`${dataAxios.msg}: ${response.config.url}`)
            break
        }
      }
    },
    error => {
      const status = get(error, 'response.status')
      switch (status) {
        case 400:
          error.message = '请求错误'
          break
        case 401:
          util.cookies.remove('token')
          util.cookies.remove('uuid')
          util.cookies.remove('refresh')
          router.push({ path: '/login' })
          error.message = '认证已失效,请重新登录~'
          break
        case 403:
          error.message = '拒绝访问'
          break
        case 404:
          error.message = `请求地址出错: ${error.response.config.url}`
          break
        case 408:
          error.message = '请求超时'
          break
        case 500:
          error.message = '服务器内部错误'
          break
        case 501:
          error.message = '服务未实现'
          break
        case 502:
          error.message = '网关错误'
          break
        case 503:
          error.message = '服务不可用'
          break
        case 504:
          error.message = '网关超时'
          break
        case 505:
          error.message = 'HTTP版本不受支持'
          break
        default:
          break
      }
      errorLog(error)
      return Promise.reject(error)
    }
  )
  return service
}

/**
 * @description 创建请求方法
 * @param {Object} service axios 实例
 */
function createRequestFunction (service) {
  // 校验是否为租户模式。租户模式把域名替换成 域名 加端口
  return function (config) {
    const token = util.cookies.get('token')
    // 进行布尔值兼容
    var params = get(config, 'params', {})
    for (const key of Object.keys(params)) {
      if (String(params[key]) === 'true') {
        params[key] = 1
      }
      if (String(params[key]) === 'false') {
        params[key] = 0
      }
    }
    const configDefault = {
      headers: {
        Authorization: 'JWT ' + token,
        'Content-Type': get(config, 'headers.Content-Type', 'application/json')
      },
      timeout: 60000,
      baseURL: util.baseURL(),
      data: {},
      params: params,
      retry: 3, // 重新请求次数
      retryDelay: 1000 // 重新请求间隔
    }
    return service(Object.assign(configDefault, config))
  }
}

// 用于真实网络请求的实例和请求方法
export const service = createService()
export const request = createRequestFunction(service)
```

6. 服务器验证登录信息，如果验证成功，返回一个包含用户信息和访问令牌（access token）的响应。

7. `login` action将访问令牌存储在cookie中，并通过`store.dispatch('d2admin/user/set', userInfoRes.data, { root: true })`将用户信息存储在Vuex store中。

```javascript
// d2admin/account.js
util.cookies.set('uuid', res.userId)
util.cookies.set('token', res.access)
util.cookies.set('refresh', res.refresh)
var userInfoRes = await request({
  url: '/api/system/user/user_info/',
  method: 'get',
  params: {}
})
await store.dispatch('d2admin/user/set', userInfoRes.data, { root: true })
```

这里需要注意下  这里的util.cookie是自定义的一个方法文件，具体是在src\libs\util.js文件中

我们去这个文件中看，确实是有cookie相关的信息操作，可见引入了另外一个js文件，即src\libs\util.cookies.js

```javascript
import cookies from './util.cookies'
```

而我们去这个方法中一看的话，果然有set的方法，其将key和value存储在js-cookie中Cookie里

```javascript
import Cookies from 'js-cookie'

const cookies = {}

/**
 * @description 存储 cookie 值
 * @param {String} name cookie name
 * @param {String} value cookie value
 * @param {Object} setting cookie setting
 */
cookies.set = function (name = 'default', value = '', cookieSetting = {}) {
  const currentCookieSetting = {
    expires: 1
  }
  Object.assign(currentCookieSetting, cookieSetting)
  Cookies.set(`d2admin-${process.env.VUE_APP_VERSION}-${name}`, value, currentCookieSetting)
}

/**
 * @description 拿到 cookie 值
 * @param {String} name cookie name
 */
cookies.get = function (name = 'default') {
  window.dvAdminToken = Cookies.get(`d2admin-${process.env.VUE_APP_VERSION}-${name}`)
  return window.dvAdminToken
}

/**
 * @description 拿到 cookie 全部的值
 */
cookies.getAll = function () {
  return Cookies.get()
}

/**
 * @description 删除 cookie
 * @param {String} name cookie name
 */
cookies.remove = function (name = 'default') {
  return Cookies.remove(`d2admin-${process.env.VUE_APP_VERSION}-${name}`)
}

export default cookies

```

这个js文件中定义了一个名为`cookies`的对象，包含一个名为`set`的方法。这个方法接收三个参数：cookie的名称、值和设置。

`cookies.set`方法的作用是存储一个cookie值。它使用了`js-cookie`库来操作cookie。具体来说，它将传入的cookie名称加上一个前缀（`d2admin-${process.env.VUE_APP_VERSION}-`），然后使用`Cookies.set`方法将cookie值存储到浏览器中。`process.env.VUE_APP_VERSION`是从环境变量中读取的Vue应用版本号，这样做的目的是避免不同版本的应用之间的cookie冲突，同时也可以避免和其他key尽可能之间尽可能保证不会重复。

`cookieSetting`参数是一个对象，用于设置cookie的属性，如过期时间、路径等。在这个方法中，默认的`cookieSetting`是`{ expires: 1 }`，表示cookie的过期时间为1天。你可以在调用`cookies.set`方法时传入自定义的设置。

以下是一个使用`cookies.set`方法的示例：

```javascript
// 设置一个名为 'username' 的cookie，值为 'John Doe'，过期时间为7天
cookies.set('username', 'John Doe', { expires: 7 });
```

8. 接下来，`login` action调用`load`方法。`load`方法在`d2admin/account`模块的Vuex store中定义。这个方法负责从持久化数据中加载一系列设置，如用户名、主题、页面过渡效果、多页列表、侧边栏配置、全局尺寸和颜色设置等，并将它们存储到Vuex store中。

```javascript
// d2admin/account.js
async load ({ dispatch }) {
  await dispatch('d2admin/user/load', null, { root: true })
  await dispatch('d2admin/theme/load', null, { root: true })
  await dispatch('d2admin/transition/load', null, { root: true })
  await dispatch('d2admin/page/openedLoad', null, { root: true })
  await dispatch('d2admin/menu/asideLoad', null, { root: true })
  await dispatch('d2admin/size/load', null, { root: true })
  await dispatch('d2admin/color/load', null, { root: true })
}
```

9. 用户登录成功后，`submit`方法将用户重定向到主页，即`this.$router.replace('/')`。

```javascript
// base.vue
async submit (formName) {
  ...
  await this.login(this.formLogin)
  this.$router.replace('/')
  ...
}
```

10. 如果服务器验证失败，`login` action会被`catch`捕获，然后在`base.vue`组件中再次调用`getCaptcha`方法获取新的验证码。

```javascript
// base.vue
async submit (formName) {
  ...
  try {
    await this.login(this.formLogin)
    this.$router.replace('/')
  } catch (error) {
    this.getCaptcha()
  }
  ...
}
```

以上就是整个登录流程的详细梳理，希望这个解释能帮助你更好地理解代码。

## 流程图

![登录验证逻辑](https://gitee.com/sugary0000/typora_image_store/raw/master/typora_images/登录验证逻辑.svg)



