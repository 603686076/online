##  前端

```vue
config.headers['Authorization'] = 'Bearer ' + getToken()
```

### `src\utils\auth.js`

```js
// 引入js-cookie
import Cookies from 'js-cookie'

// 设置Token，1天,与后端同步
export function setToken(token) {
    return Cookies.set(uToken, token, {expires: 1})
}
```

### `src\views\auth\Login.vue`

```vue
<template>
  <div class="columns py-6">
    <div class="column is-half is-offset-one-quarter">
      <el-card shadow="never">
        <div slot="header" class="has-text-centered has-text-weight-bold">
          用户登录
        </div>
        <div>
          <el-form
            v-loading="loading"
            :model="ruleForm"
            status-icon
            :rules="rules"
            ref="ruleForm"
            label-width="100px"
            class="demo-ruleForm"
          >
            <el-form-item label="账号" prop="name">
              <el-input v-model="ruleForm.name"></el-input>
            </el-form-item>

            <el-form-item label="密码" prop="pass">
              <el-input
                type="password"
                v-model="ruleForm.pass"
                autocomplete="off"
              ></el-input>
            </el-form-item>

            <el-form-item label="记住" prop="delivery">
              <el-switch v-model="ruleForm.rememberMe"></el-switch>
            </el-form-item>

            <el-form-item>
              <el-button type="primary" @click="submitForm('ruleForm')"
                >提交</el-button
              >
              <el-button @click="resetForm('ruleForm')">重置</el-button>
            </el-form-item>
          </el-form>
        </div>
      </el-card>
    </div>
  </div>
</template>

<script>
export default {
  name: "Login",
  data() {
    return {
      redirect: undefined,
      loading: false,
      ruleForm: {
        name: "",
        pass: "",
        rememberMe: true,
      },
      rules: {
        name: [
          { required: true, message: "请输入账号", trigger: "blur" },
          {
            min: 2,
            max: 15,
            message: "长度在 2 到 15 个字符",
            trigger: "blur",
          },
        ],
        pass: [
          { required: true, message: "请输入密码", trigger: "blur" },
          {
            min: 6,
            max: 20,
            message: "长度在 6 到 20 个字符",
            trigger: "blur",
          },
        ],
      },
    };
  },
  methods: {
    submitForm(formName) {
      this.$refs[formName].validate((valid) => {
        if (valid) {
          this.loading = true;
          this.$store
            .dispatch("user/login", this.ruleForm) // 调用异步请求
            .then(() => {
              this.$message({
                message: "恭喜你，登录成功",
                type: "success",
                duration: 2000,
              });

              setTimeout(() => {
                this.loading = false;
                this.$router.push({ path: this.redirect || "/" });
              }, 0.1 * 1000);
            })
            .catch(() => {
              this.loading = false;
            });
        } else {
          return false;
        }
      });
    },
    resetForm(formName) {
      this.$refs[formName].resetFields();
    },
  },
};
</script>

<style scoped>
</style>
```





### `src\store\modules\user.js`

```js
import { getUserInfo, login, logout } from "@/api/auth/auth";
import { getToken, setToken, removeToken } from "@/utils/auth";

// 定义全局数据
const state = {
  token: getToken(), // token
  user: "", // 用户对象
};

// state的状态改变必须通过mutations进行，同步更新state中的数据
const mutations = {
  SET_TOKEN_STATE: (state, token) => {
    state.token = token;
  },
  SET_USER_STATE: (state, user) => {
    state.user = user;
  },
};

// 异步，通知mutations更新state中的数据
const actions = {
  // 用户登录
  login({ commit }, userInfo) { // 响应异步请求，进行操作
    console.log(userInfo);
    const { name, pass, rememberMe } = userInfo;
    return new Promise((resolve, reject) => {
      login({ username: name.trim(), password: pass, rememberMe: rememberMe })
        .then((response) => { // 后端返回的响应体
          const { data } = response;
          commit("SET_TOKEN_STATE", data.token); // 在state中设置token
          setToken(data.token); // 在js-cookie中设置token
          resolve();
        })
        .catch((error) => {
          reject(error);
        });
    });
  },
  // 获取用户信息
  getInfo({ commit, state }) {
    return new Promise((resolve, reject) => {
      getUserInfo()
        .then((response) => {
          const { data } = response;
          if (!data) {
            commit("SET_TOKEN_STATE", "");
            commit("SET_USER_STATE", "");
            removeToken();
            resolve();
            reject("Verification failed, please Login again.");
          }
          commit("SET_USER_STATE", data);
          resolve(data);
        })
        .catch((error) => {
          reject(error);
        });
    });
  },
  // 注销
  logout({ commit, state }) {
    return new Promise((resolve, reject) => {
      logout(state.token)
        .then((response) => {
          console.log(response);
          commit("SET_TOKEN_STATE", "");
          commit("SET_USER_STATE", "");
          removeToken();
          resolve();
        })
        .catch((error) => {
          reject(error);
        });
    });
  },
};

export default {
  namespaced: true,
  state,
  mutations,
  actions,
};

```





### js-cookie

token保存在浏览器的Cookies里

![image-20210521184002205](JWT/image-20210521184002205.png)

## 后端

1. 启动类`DoubaoApplication`

   ```java
   @Bean
   public FilterRegistrationBean jwtFilter() {
       final FilterRegistrationBean registrationBean = new FilterRegistrationBean();
       JwtAuthenticationFilter filter = new JwtAuthenticationFilter();
       registrationBean.setFilter(filter);
       return registrationBean;
   }
   ```

2. 工具类`JwtAuthenticationFilter`

   ```java
   package com.douyuehan.doubao.jwt;
   
   import org.springframework.util.AntPathMatcher;
   import org.springframework.util.PathMatcher;
   import org.springframework.web.filter.OncePerRequestFilter;
   
   import javax.servlet.FilterChain;
   import javax.servlet.ServletException;
   import javax.servlet.http.HttpServletRequest;
   import javax.servlet.http.HttpServletResponse;
   import java.io.IOException;
   import java.util.ArrayList;
   import java.util.List;
   
   
   public class JwtAuthenticationFilter extends OncePerRequestFilter {
       private static final PathMatcher pathMatcher = new AntPathMatcher();
   
       @Override
       protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
   
           try {
               // 检查是否为受保护的请求
               if(isProtectedUrl(request)) {
   				// System.out.println(request.getMethod());
                   // 哨兵请求
                   if(!request.getMethod().equals("OPTIONS"))
                       // 校验token
                       request = JwtUtil.validateTokenAndAddUserIdToHeader(request);
               }
           } catch (Exception e) {
               response.sendError(HttpServletResponse.SC_UNAUTHORIZED, e.getMessage());
               return;
           }
           filterChain.doFilter(request, response);
       } 
   
       
       // 定义要拦截的请求
       private boolean isProtectedUrl(HttpServletRequest request) {
           List<String> protectedPaths = new ArrayList<String>();
           protectedPaths.add("/ums/user/info");
           protectedPaths.add("/ums/user/update");
           protectedPaths.add("/post/create");
           protectedPaths.add("/post/update");
           protectedPaths.add("/post/delete/*");
           protectedPaths.add("/comment/add_comment");
           protectedPaths.add("/relationship/subscribe/*");
           protectedPaths.add("/relationship/unsubscribe/*");
           protectedPaths.add("/relationship/validate/*");
   
           boolean bFind = false;
           for( String passedPath : protectedPaths ) {
               bFind = pathMatcher.match(passedPath, request.getServletPath());
               if( bFind ) {
                   break;
               }
           }
           return bFind;
       }
   
   }
   ```

3. JWT工具类`src/main/java/com/douyuehan/doubao/jwt/JwtUtil.java`

   ```java
   public static HttpServletRequest validateTokenAndAddUserIdToHeader(HttpServletRequest request) {
       // 从请求头里拿到token
       String token = request.getHeader(HEADER_STRING);
       if (token != null) {
           // parse the token.
           try {
               Map<String, Object> body = Jwts.parser()
                   .setSigningKey(SECRET)
                   .parseClaimsJws(token.replace(TOKEN_PREFIX, ""))
                   .getBody();
               return new CustomHttpServletRequest(request, body);
           } catch (Exception e) { 
               logger.info(e.getMessage());
               throw new TokenValidationException(e.getMessage());
           }
       } else {
           throw new TokenValidationException("Missing token");
       }
   }
   ```

## 整个流程

前端发来请求：

> General
>
> ​	Request URL:  http://47.104.252.157:7006/ums/user/info
>
> ​	Request Method: GET
>
> ​	Status Code:  200 
>
> 
>
> Request Headers 
>
> ​	Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJ1c2VyTmFtZSI6Inh1IiwiZXhwIjoxNjI1MzY5MjUwfQ.V6SyqTrItIZZx0PBzdJ0zIjHstDZHBknFlC0P1GH_r6rK8hlZY3rLFn7hNwTzwWj9Q90R-cT63g2Zzj0AsTgjg
>
> ​	Connection: keep-alive
>
> ​	Host: 47.104.252.157 :7006

拦截器拦截请求，通过 `request = JwtUtil.validateTokenAndAddUserIdToHeader(request);` 修改请求

通过`String token = request.getHeader(HEADER_STRING);`(`HEADER_STRING = "Authorization"`)，getHeader方法获取

Authorization，即token ->`Bearer eyJhbGciOiJIUzUxMiJ9.eyJ1c2VyTmFtZSI6Inh1IiwiZXhwIjoxNjI1MzY5MjUwfQ.V6SyqTrItIZZx0PBzdJ0zIjHstDZHBknFlC0P1GH_r6rK8hlZY3rLFn7hNwTzwWj9Q90R-cT63g2Zzj0AsTgjg`

解析之后，得到`Map<String, String>`类型的对象，里面是之前生成token（`map.put(USER_NAME, userName);`）时候的放入的对象

`USER_NAME = "userName";`即` userName->xu` 键值对

这个时候我们自定义`CustomHttpServletRequest extends HttpServletRequestWrapper`类，重写`getHeaders`方法

controller层通过注解`@RequestHeader(value = USER_NAME)`，会调用重写后的`getHeaders`方法，根据`USER_NAME`，即`userName`返回`xu`

这样就实现了客户端通过get方法在请求头里携带`Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJ1c2VyTmFtZSI6Inh1IiwiZXhwIjoxNjI1MzY5MjUwfQ.V6SyqTrItIZZx0PBzdJ0zIjHstDZHBknFlC0P1GH_r6rK8hlZY3rLFn7hNwTzwWj9Q90R-cT63g2Zzj0AsTgjg`这个token，后端就可以通过这个token解析出来用户名。



**简而言之**

需要身份认证的请求会被后端的`JwtAuthenticationFilter`拦截，它会更改请求。首先会检查请求头里的token，有token的话对token进行解析，解析出来曾经的加密对象 `userName -> xu`，再通过这个加密对象对原请求进行更改，令原请求`request` = 新创建的 `Http Servlet` 请求，达到修改的目的。新请求会重写 `getHeaders(String name)`方法，这样`UmsUserController`就能通过`@RequestHeader(value = USER_NAME)`注解的方式调用`request`的方法，我们刚刚对这个请求的 `getHeaders(String name)`方法进行了更改，则会调用我们重写的方法从而获取到用户名`xu`。于是就实现了从前端发来的请求体里的token中解析出用户名，进而进行下一步的操作。

