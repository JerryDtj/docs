# shiro——再看shiro

上篇大概介绍了shiro的三大主体：subject(主体)、Realm(存储介质)、SecurityManager(权限管理)。

这篇将围绕这三大主题，介绍下三大主体的作用，以及他们是怎么样协同工作的。

最后会附带一个和springBoot整合的示例(可运行)。让各位同学可以通过代码对shiro有个更加直观的认识。

废话不说，正文开始

## 一、subject 接口

先上源码，看看shiro团队是怎么说的

```java
/**
 * {@code Subject}表示单个应用程序用户的状态和安全操作。
 * 这些操作包括身份验证（登录/注销），授权（访问控制）和会话访问。这是Shiro的单用户安全功能的主要机制。
 * <h3>获得主题</h3>
 * 要获得当前正在执行的Subject，应用程序开发人员几乎总是会使用 SecurityUtils
 * org.apache.shiro.SecurityUtils#getSubject() 
 * 几乎所有安全操作都应该使用此方法返回的{@code Subject}执行。
 * <h3>许可方法</h3>
 * 请注意，此接口中有许多Permission方法被重载以接受String参数而不是Permission实例。
 * 如果需要，它们是一种便利，允许调用者使用一个Permission的字符串表示。基础授权子系统实现通常
 * 只是简单地将这些String值转换为Permission实例，然后只调用相应的type-safe方法。 
 *（Shiro的默认实现使用org.apache.shiro.authz.permission.PermissionResolver
 * 对这些方法执行字符串到权限的转换。）
 * <p/>
 * @since 0.1
 */
public interface Subject {

    /** 如果此Subject是匿名的，则返回此Subject的应用程序范围的唯一标识主体，或{@code null}，
      * 因为它还没有任何关联的帐户数据（例如，如果他们尚未登录） 。 
      * <p /> 
      * 获取用户信息
     */
    Object getPrincipal();
    PrincipalCollection getPrincipals();
    boolean isPermitted(String permission);
	boolean isPermitted(Permission permission);
	boolean[] isPermitted(String... permissions);
    boolean[] isPermitted(List<Permission> permissions);
    boolean isPermittedAll(String... permissions);
    boolean isPermittedAll(Collection<Permission> permissions);
    void checkPermission(String permission) throws AuthorizationException;
    void checkPermission(Permission permission) throws AuthorizationException;
    void checkPermissions(String... permissions) throws AuthorizationException;
    void checkPermissions(Collection<Permission> permissions) throws AuthorizationException;
    boolean hasRole(String roleIdentifier);
    boolean[] hasRoles(List<String> roleIdentifiers);
    boolean hasAllRoles(Collection<String> roleIdentifiers);
    void checkRole(String roleIdentifier) throws AuthorizationException;
    void checkRoles(Collection<String> roleIdentifiers) throws AuthorizationException;
    void checkRoles(String... roleIdentifiers) throws AuthorizationException
    /**
      * 对此主题/用户执行登录尝试。如果不成功，则抛出AuthenticationException，
      * 其子类标识尝试失败的原因。 如果成功，与提交的主体/凭证相关联的帐户数据将与此Subject相关联，
      * 并且该方法将安静地返回。 
      * <p /> 
      * 静默返回后，可以认为此Subject实例已经过身份验证，getPrincipal（）将为非null
      * 且isAuthenticated()将是true。
     **/
    void login(AuthenticationToken token) throws AuthenticationException;
    boolean isAuthenticated();
	/**
	  * 如果此Subject具有标识(它不是匿名的)并且在前一个会话期间从成功的身份验证中记住身份则返回 true。 		 * <p /> 
	  * 尽管底层实现确切地确定了该方法的功能，但大多数实现都使用此方法作为此代码的逻辑等效项：
	  * getPrincipal（）}！= null &&！isAuthenticated（）
      * <p /> 
      * 请注意，如上面的代码示例所示，如果记住Subject，则它们是被视为不通过的身份验证。
      * 对isAuthenticated（）的检查是比此方法反映的更严格的检查。例如，检查主题是否可以访问财务
      * 信息应该几乎总是依赖于isAuthenticated（）来保证已验证的身份，而不是此方法。 
      * <p /> 
      *一旦主体经过身份验证，就不再仅仅考虑它们，因为它们的身份将在当前会话期间得到验证。 
      * <h4>记住与经过身份验证</ h4> 
      * 身份验证是证明的过程，您就是您所说的人。当只记住用户时，记住的身份让系统知道该用户可能是谁，
      * 但实际上，如果记住的Subject代表的话，则无法绝对保证用户当前使用该应用程序。 
      * <p /> 
      * 因此，尽管应用程序的许多部分仍然可以基于记住的principalals执行特定于用户的逻辑，
      * 例如自定义视图，但它应该永远不会执行高度敏感的
      * 操作用户通过执行成功的身份验证尝试合法地验证了他们的身份。
      * <p /> 
      * 我们在整个网络上看到这种模式，我们将使用
      * <a href="http://www.amazon.com"> Amazon.com </a>作为示例：
      * <p /> 
      * 当您访问Amazon.com并执行登录并要求其“记住我”时，它将设置一个带有您*身份的cookie。
      * 如果您没有注销并且您的会话过期，并且您回来了，请说第二天，亚马逊仍然知道您可能的人：
      * 您仍然可以看到您的所有书籍和电影推荐以及类似内容特定于用户的功能，
      * 因为它们基于您（记住的）用户ID。但是，如果您尝试执行某些敏感操作，例如访问您帐户的结算数据，
      * 亚马逊强制您进行实际登录，需要您的用户名和密码。 
      * <p /> 
      * 这是因为尽管amazon.com从“记住我”中获取了您的身份，但它承认您并未经过
      * 实际身份验证。真正保证您的唯一方法就是强制您执行实际成功的身份验证，这是您说出来的人，
      * 因此允许您访问敏感帐户数据。您可以通过isAuthenticated（）方法检查此保证，而不是通过此方法。
	 **/
    boolean isRemembered();
	/**
	  * 返回与此Subject关联的应用程序Session。如果调用此方法时不存在会话，
	  * 则将创建与此Subject关联的新会话，然后返回。
	  * 注意此session非httpSession。此session可以不依靠web，
	  * 他是shiro团队自定义出来的一个类似于httpSession的东西。
	 **/
    Session getSession(boolean create);

    void logout();
	<V> V execute(Callable<V> callable) throws ExecutionException;
    void execute(Runnable runnable);
    <V> Callable<V> associateWith(Callable<V> callable);
    Runnable associateWith(Runnable runnable);
	void runAs(PrincipalCollection principals) throws NullPointerException, IllegalStateException;
	boolean isRunAs();
    PrincipalCollection getPreviousPrincipals();
    PrincipalCollection releaseRunAs();
    /**
      * 构建器设计模式实现，用于以简化的方式创建{@link Subject}实例，而无需了解Shiro的构造技术。 
      * <p /> 
      * 注意：这仅用于框架开发支持，通常不应由应用程序开发人员使用。通常应使用
      * <code> SecurityUtils.getSubject（）获取Subject实例。
      * <h4>用法</ h4> 
      *此构建器的最简单用法是构建一个匿名的，无会话的{@code Subject}实例：
      * <pre> * Subject subject = new Subject.Builder（）。
      * {@ link #buildSubject（）buildSubject（）} ; 
      * 上面显示的默认的无参Subject.Builder（）}构造函数将通过 SecurityUtils使用应用程序的
      * 当前可访问的SecurityManager。
      * {@ link SecurityUtils＃getSecurityManager（ ）getSecurityManager（）} 
      * 您还可以指定其他Subject使用的确切{@code SecurityManager}实例。
      * {@ link #Builder（org.apache.shiro.mgt.SecurityManager）Builder（securityManager）} 	  * 构造函数，如果需要。 
      * <p /> 
      * 可以在buildSubject（）方法之前调用所有其他方法，以提供有关如何构造{@code Subject}实例的
      * 上下文。例如，如果您有会话ID并且想要获得“拥有”该会话的Subject（假设会话存在且未过期）：
      * <pre> 
      * Subject subject = new Subject.Builder（ ）.sessionId（sessionId）.buildSubject（）; 		* 同样，如果你想要一个反映某个身份的{@code Subject}实例：
      * Subject subject = new Subject.Builder().sessionId(sessionId).buildSubject()
      * 注意:返回的{@code Subject}实例是不会自动绑定到应用程序（线程以供进一步使用。也就是说，
      * SecurityUtilsgetSubject（）不会自动返回与构建器返回的实例相同的实例。框架
      * 开发人员可以根据需要绑定构建的{@code Subject}以继续使用。
    **/
	public static class Builder {
        private final SubjectContext subjectContext;
        private final SecurityManager securityManager;
        public Builder() {
            this(SecurityUtils.getSecurityManager());
        }
        public Builder(SecurityManager securityManager) {
            if (securityManager == null) {
                throw new NullPointerException("SecurityManager method argument cannot be null.");
            }
            this.securityManager = securityManager;
            this.subjectContext = newSubjectContextInstance();
            if (this.subjectContext == null) {
                throw new IllegalStateException("Subject instance returned from 'newSubjectContextInstance' " +
                        "cannot be null.");
            }
            this.subjectContext.setSecurityManager(securityManager);
        }
        protected SubjectContext newSubjectContextInstance() {
            return new DefaultSubjectContext();
        }
        protected SubjectContext getSubjectContext() {
            return this.subjectContext;
        }
        public Builder sessionId(Serializable sessionId) {
            if (sessionId != null) {
                this.subjectContext.setSessionId(sessionId);
            }
            return this;
        }
        public Builder host(String host) {
            if (StringUtils.hasText(host)) {
                this.subjectContext.setHost(host);
            }
            return this;
        }
        public Builder session(Session session) {
            if (session != null) {
                this.subjectContext.setSession(session);
            }
            return this;
        }
        public Builder principals(PrincipalCollection principals) {
            if (!CollectionUtils.isEmpty(principals)) {
                this.subjectContext.setPrincipals(principals);
            }
            return this;
        }

        public Builder sessionCreationEnabled(boolean enabled) {
            this.subjectContext.setSessionCreationEnabled(enabled);
            return this;
        }

        public Builder authenticated(boolean authenticated) {
            this.subjectContext.setAuthenticated(authenticated);
            return this;
        }

        public Builder contextAttribute(String attributeKey, Object attributeValue) {
            if (attributeKey == null) {
                String msg = "Subject context map key cannot be null.";
                throw new IllegalArgumentException(msg);
            }
            if (attributeValue == null) {
                this.subjectContext.remove(attributeKey);
            } else {
                this.subjectContext.put(attributeKey, attributeValue);
            }
            return this;
        }
		/**
		 * 创建并返回一个新的{@code Subject}实例，该实例反映此类中其他方法获取的累积状态。 
		 * <p /> 
		 * 调用此方法后，此{@code Builder}实例仍将保留基础状态 - 新创建的它不会清除已存在的它;
		 * 重复调用此方法将返回多个{@link Subject}实例，所有
		 *都反映完全相同的状态。如果要构造新的（不同的）{@code Subject}，则必须创建新的
		 * {@code Builder}实例。 
		 * <p /> 
		 * 注意返回的{@code Subject}实例不自动绑定到应用程序（线程）以供进一步使用。也就是说，
		 * SecurityUtils#getSubject不会自动返回与构建器返回的实例相同的实例。 
		 * 框架开发人员可以根据需要绑定返回的{@code Subject}以继续使用。 
		 * 
		 * @return一个新的{@code Subject}实例，反映了此类中其他方法获取的累积状态。
		**/
        public Subject buildSubject() {
            return this.securityManager.createSubject(this.subjectContext);
        }
    }

}

```

从源码上可以看出，subject 主要包含以下东西

- getPrincipal（） 登录用户信息
- hasRole（）是否有权限
- login（） 用户登录
- isRemembered（）是否记住我
- isAuthenticated（）是否通过验证
- getSession（）一些用细信息存储‘’
- logout（）登出
- Builder（）subject 初始化的一些方法

从Builder()方法上可以看出subject的获得方式为：SecurityUtils.getSubject()：

那SecurityUtils又为何许人也，请接着往下看。

## 二、SecurityUtils

SecurityUtils源码如下：

```java
public abstract class SecurityUtils {
	/**
	* 声明一个私有的全局惟一的SecurityManager。
	**/
    private static SecurityManager securityManager;

    /**
     * 根据运行时环境返回调用代码可用的当前可访问的Subject
     * <p/>
     * 提供此方法是为了获得{@code Subject}而无需借助*特定于实现的方法。
     * 它还允许Shiro团队在将来根据需求/更新更改此方法的基础实现，而不会影响使用它的代码
     *
     * @return 当前可访问的{@code Subject}可供调用代码访问。
     * @throws IllegalStateException 如果没有Subject实例或 Subject 中没有SecurityManager实例 	  *                               会抛出以这个异常，Subject应该总是可供调用者使用。
     */
    public static Subject getSubject() {
        Subject subject = ThreadContext.getSubject();
        if (subject == null) {
            subject = (new Subject.Builder()).buildSubject();
            ThreadContext.bind(subject);
        }
        return subject;
    }

   	/**
   	* 设置VM（静态）单例SecurityManager，专门用于透明使用
   	* */
    public static void setSecurityManager(SecurityManager securityManager) {
        SecurityUtils.securityManager = securityManager;
    }

   /**
   * 返回调用代码可访问的SecurityManager。
   * 这个实现有利于获取线程绑定的SecurityManager，如果它可以找到一个。如果一个
   * 不可用于执行线程，它将尝试使用静态单例（如果可用）.
   * 如果线程局部实例或静态单例实例都不可用，则此方法抛出
   * {@code UnavailableSecurityManagerException}以指示错误 
   * - 应始终可以访问SecurityManager以调用应用程序中的代码。如果不是，则可能是由于Shiro配置问题。
   * @throws UnavailableSecurityManagerException
   **/
    public static SecurityManager getSecurityManager() throws UnavailableSecurityManagerException {
        SecurityManager securityManager = ThreadContext.getSecurityManager();
        if (securityManager == null) {
            securityManager = SecurityUtils.securityManager;
        }
        if (securityManager == null) {
            String msg = "No SecurityManager accessible to the calling code, either bound to the " + ThreadContext.class.getName() + " or as a vm static singleton.  This is an invalid application " + "configuration.";
            throw new UnavailableSecurityManagerException(msg);
        }
        return securityManager;
    }
}
```

在SecurityUtils中，维护了一个SecurityManager，然后调用(new Subject.Builder()).buildSubject() 来返回一个单例subject对象。*~~里面设计模式模式什么就不说了，不是本文的重点，喜欢钻研的同学可以参考 《设计模式之禅》或《大话设计模式》。~~*

## 三、SecurityManager

通过SecurityUtils 有牵扯出来了shiro第二大对象SecurityManager，让我们一起看看SecurityManager里面有什么东西吧，具体的源码如下：

```java
/**
* SecurityManager}跨单个应用程序执行所有主题的所有安全操作。 
* <p /> 
* 界面本身主要是为了方便起见 - 它扩展了{@link org.apache.shiro.authc.Authenticator}，
* {@link Authorizer}和{@link SessionManager}接口，从而巩固了
* 这些行为成为单一的参考点。对于大多数Shiro用法，这简化了配置，并且
* 往往比分别引用{@code Authenticator}，{@code Authorizer}和
* {@code SessionManager}实例更方便;相反，只需要与单个
* {@code SecurityManager}实例进行交互。 
* <p /> 
* 除了上述三个接口外，此接口还提供了许多支持
* {@link Subject}行为的方法。 
* {@link org.apache.shiro.subject.Subject Subject}为<em>单个</ em>用户执行
* 身份验证，授权和会话操作，因此只能由{@code A SecurityManager管理知道所有三个功能。 
* 另一方面，三个父接口不“了解”{@code Subject}，以确保关注点的清晰分离。 
* <p /> 
* <b>使用说明</ b>：实际上，绝大多数应用程序员都不会经常与SecurityManager 
* 交互，如果有的话。 <em>大多数</ em>应用程序程序员只关心当前*执行用户的安全操作，通常通过调用
* {@link org.apache.shiro.SecurityUtils＃getSubject（）SecurityUtils.getSubject（）}来实现。 * <p /> 
* 另一方面，框架开发人员可能会发现使用实际的SecurityManager非常有用。 
*
**/
public interface SecurityManager extends Authenticator, Authorizer, SessionManager {
    Subject login(Subject var1, AuthenticationToken var2) throws AuthenticationException;
    void logout(Subject var1);
    /**
    * 创建反映指定上下文数据的{@code Subject}实例。 
    * <p /> 
    *上下文可以是此{@code SecurityManager}构建{@code Subject}实例所需的任何内容。 
    * 大多数Shiro最终用户永远不会调用此方法 - 它主要用于*框架开发，
    * 并支持{@code SecurityManager}可能使用的任何基础自定义
    * {@link SubjectFactory SubjectFactory}实现。 
    * <h4>用法</ h4> 
    * 调用此方法后，返回的实例<em>不</ em>绑定到应用程序以供进一步使用。 
    * 调用者应该知道{@code Subject}实例仅具有本地范围，并且必须明确管理调用方法之外
    * 的任何其他进一步使用。
    **/
    Subject createSubject(SubjectContext var1);
}
```

SecurityManager对象主要包含 三个方法，1个登录，一个登录 还有一个创建主题的方法。

看到这里有可能有眼尖的小伙伴会问SecurityManager 不是包含subject 嘛，为什么这里也有一个login方法。

## 四、subject实现

![1562211829566](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1562211829566.png)

在DelegatingSubject中login实现类如下：

![1562211921466](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1562211921466.png)

可以看到登录需要先传一个AuthenticationToken对象。所以也需要自己构建下UsernamePasswordToken对象

![1562212078686](C:\Users\pc\AppData\Roaming\Typora\typora-user-images\1562212078686.png)

从三大主体源码可以看出，要使用shiro，需要初始化以下对象：

- Realm

- SecurityManager

- ShiroFilterFactoryBean

- UsernamePasswordToken

  这些东西可以用spring做统一管理。

  源码看到这里，基本上大体的架构就清楚了，来让我们一起实战吧。

## 五、shiro实战

- 
  具体的实现源码请参考：http://47.93.119.76:3000/dengtj/ShiroSpringBootDemo.git





