只看<module>oauth2-logout</module>就行，用的是Gateway and Internal Authserver (GIA).
架构是前面一个gateway，后边一个认证server，一个resource server，一个UI server



但代码用的不是最好的方式With this simple change, as soon as you authenticate, the session in the authserver is already dead, so there’s no need to try and manage it from the client. When you log out of the UI app, and then log back in, the authserver doesn’t recognize you and prompts for credentials. This pattern is the one implemented by the oauth2-logout sample in the source code for this tutorial. The downside of this approach is that you don’t really have true single sign on any more - any other apps that are part of your system will find that the authserver session is dead and they have to prompt for authentication again - it isn’t a great user experience if there are multiple apps.

这种写法有些激进了，

更好的方式是取消这一步，移除
@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints)
    throws Exception {
  ...
  endpoints.addInterceptor(new HandlerInterceptorAdapter() {
    @Override
    public void postHandle(HttpServletRequest request,
        HttpServletResponse response, Object handler,
        ModelAndView modelAndView) throws Exception {
      if (modelAndView != null
          && modelAndView.getView() instanceof RedirectView) {
        RedirectView redirect = (RedirectView) modelAndView.getView();
        String url = redirect.getUrl();
        if (url.contains("code=") || url.contains("error=")) {
          HttpSession session = request.getSession(false);
          if (session != null) {
            session.invalidate();
          }
        }
      }
    }
  });
}

具体的参见原文档，建议选用Logout of Both Servers from Browser这一节的方式



--------------------
-------------------
下面的内容摘自另一个教程，供参考
login、logout都被@EnableOAuth2Sso处理了

Spring Security has built in support for a /logout endpoint which will do the right thing for us (clear the session and invalidate the cookie).

The /logout endpoint requires us to POST to it, and to protect the user from Cross Site Request Forgery (CSRF, pronounced "sea surf"), it requires a token to be included in the request. The value of the token is linked to the current session, which is what provides the protection, so we need a way to get that data into our JavaScript app.

Many JavaScript frameworks have built in support for CSRF (e.g. in Angular they call it XSRF), but it is often implemented in a slightly different way than the out-of-the box behaviour of Spring Security. For instance in Angular the front end would like the server to send it a cookie called "XSRF-TOKEN" and if it sees that, it will send the value back as a header named "X-XSRF-TOKEN". We can implement the same behaviour with our simple jQuery client, and then the server side changes will work with other front end implementations with no or very few changes. To teach Spring Security about this we need to add a filter that creates the cookie and also we need to tell the existing CRSF filter about the header name.

--------------
How to Add a Local User Database
Many applications need to hold data about their users locally, even if authentication is delegated to an external provider. We don’t show the code here, but it is easy to do in two steps.

Choose a backend for your database, and set up some repositories (e.g. using Spring Data) for a custom User object that suits your needs and can be populated, fully or partially, from the external authentication.

Provision a User object for each unique user that logs in by inspecting the repository in your /user endpoint. If there is already a user with the identity of the current Principal, it can be updated, otherwise created.

Hint: add a field in the User object to link to a unique identifier in the external provider (not the user’s name, but something that is unique to the account in the external provider).
---------

@Override
protected void configure(HttpSecurity http) throws Exception {
  http.antMatcher("/**")                                       (1)
    .authorizeRequests()
      .antMatchers("/", "/login**", "/webjars/**").permitAll() (2)
      .anyRequest().authenticated()                            (3)
    .and().exceptionHandling()
      .authenticationEntryPoint(new LoginUrlAuthenticationEntryPoint("/")) (4)
    ...
}
1 All requests are protected by default
2 The home page and login endpoints are explicitly excluded
3 All other endpoints require an authenticated user
4 Unauthenticated users are re-directed to the home page
