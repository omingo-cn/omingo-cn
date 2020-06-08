---
title: keycloak-用户session数量限制
date: 2020-06-08 13:40:52
categories: Keycloak
tags:
- keycloak
---

# session并发限制
有时候需要限制用户同时在线的数量，就像微信一样同一时间只能有一个手机能够登陆上。

## 代码
废话不多说，直接放码,要的拿去。

`UserSessionLimitsAuthenticatorFactory.java`

```java 

@AutoService(AuthenticatorFactory.class)
public class UserSessionLimitsAuthenticatorFactory implements AuthenticatorFactory {

  public static final String USER_REALM_LIMIT = "userRealmLimit";
  public static final String USER_CLIENT_LIMIT = "userClientLimit";
  public static final String BEHAVIOR = "behavior";
  public static final String DENY_NEW_SESSION = "Deny new session";
  public static final String TERMINATE_OLDEST_SESSION = "Terminate oldest session";
  public static final String USER_SESSION_LIMITS = "user-session-limits";


  private static final AuthenticationExecutionModel.Requirement[] REQUIREMENT_CHOICES = {
    AuthenticationExecutionModel.Requirement.REQUIRED,
    AuthenticationExecutionModel.Requirement.DISABLED
  };

  @Override
  public String getDisplayType() {
    return "User session count limiter";
  }

  @Override
  public String getReferenceCategory() {
    return null;
  }

  @Override
  public boolean isConfigurable() {
    return true;
  }

  @Override
  public AuthenticationExecutionModel.Requirement[] getRequirementChoices() {
    return REQUIREMENT_CHOICES.clone();
  }

  @Override
  public boolean isUserSetupAllowed() {
    return false;
  }

  @Override
  public String getHelpText() {
    return "Configures how many concurrent sessions a single user is allowed to create for this realm and/or client";
  }

  @Override
  public List<ProviderConfigProperty> getConfigProperties() {
    ProviderConfigProperty userRealmLimit = new ProviderConfigProperty();
    userRealmLimit.setName(USER_REALM_LIMIT);
    userRealmLimit.setLabel("Maximum concurrent sessions for each user");
    userRealmLimit.setType(ProviderConfigProperty.STRING_TYPE);

    ProviderConfigProperty userClientLimit = new ProviderConfigProperty();
    userClientLimit.setName(USER_CLIENT_LIMIT);
    userClientLimit.setLabel("Maximum concurrent sessions for each user per keycloak client");
    userClientLimit.setType(ProviderConfigProperty.STRING_TYPE);

    ProviderConfigProperty behaviourProperty = new ProviderConfigProperty();
    behaviourProperty.setName(BEHAVIOR);
    behaviourProperty.setLabel("Behavior when user session limit is exceeded");
    behaviourProperty.setType(ProviderConfigProperty.LIST_TYPE);
    behaviourProperty.setDefaultValue(DENY_NEW_SESSION);
    behaviourProperty.setOptions(Arrays.asList(DENY_NEW_SESSION, TERMINATE_OLDEST_SESSION));

    return Arrays.asList(userRealmLimit, userClientLimit, behaviourProperty);
  }

  @Override
  public Authenticator create(KeycloakSession keycloakSession) {
    return new UserSessionLimitsAuthenticator(keycloakSession);
  }

  @Override
  public void init(Config.Scope scope) {
    // Do nothing
  }

  @Override
  public void postInit(KeycloakSessionFactory keycloakSessionFactory) {
    // Do nothing
  }

  @Override
  public void close() {
    // Do nothing
  }

  @Override
  public String getId() {
    return USER_SESSION_LIMITS;
  }
}

```
`AbstractSessionLimitsAuthenticator.java`
```java
public abstract class AbstractSessionLimitsAuthenticator implements Authenticator {

  protected KeycloakSession session;

  protected boolean exceedsLimit(long count, long limit) {
    if (limit < 0) { // if limit is negative, no valid limit configuration is found
      return false;
    }
    return count > limit - 1;
  }

  protected int getIntConfigProperty(String key, Map<String, String> config) {
    String value = config.get(key);
    if (StringUtils.isBlank(value)) {
      return -1;
    }
    return Integer.parseInt(value);
  }

  @Override
  public void action(AuthenticationFlowContext context) {

  }

  @Override
  public boolean requiresUser() {
    return false;
  }

  @Override
  public boolean configuredFor(KeycloakSession session, RealmModel realm, UserModel user) {
    return true;
  }

  @Override
  public void setRequiredActions(KeycloakSession session, RealmModel realm, UserModel user) {
  }

  @Override
  public void close() {

  }
}

```

`UserSessionLimitsAuthenticator.java`
```java 
@JBossLog
public class UserSessionLimitsAuthenticator extends AbstractSessionLimitsAuthenticator {

  String behavior;

  public UserSessionLimitsAuthenticator(KeycloakSession session) {
    this.session = session;
  }

  @Override
  public void authenticate(AuthenticationFlowContext context) {
    AuthenticatorConfigModel authenticatorConfig = context.getAuthenticatorConfig();
    Map<String, String> config = authenticatorConfig.getConfig();

    // Get the configuration for this authenticator
    behavior = config.get(UserSessionLimitsAuthenticatorFactory.BEHAVIOR);
    int userRealmLimit = getIntConfigProperty(
      UserSessionLimitsAuthenticatorFactory.USER_REALM_LIMIT, config);
    int userClientLimit = getIntConfigProperty(
      UserSessionLimitsAuthenticatorFactory.USER_CLIENT_LIMIT, config);

    if (context.getRealm() != null && context.getUser() != null) {

      // Get the session count in this realm for this specific user
      List<UserSessionModel> userSessionsForRealm = session.sessions()
        .getUserSessions(context.getRealm(), context.getUser());
      int userSessionCountForRealm = userSessionsForRealm.size();

      // Get the session count related to the current client for this user
      ClientModel currentClient = context.getAuthenticationSession().getClient();
      log.infof("session-limiter's current keycloak clientId: %s", currentClient.getClientId());

      List<UserSessionModel> userSessionsForClient = userSessionsForRealm.stream().filter(
        session -> session.getAuthenticatedClientSessionByClient(currentClient.getId()) != null)
        .collect(Collectors.toList());
      int userSessionCountForClient = userSessionsForClient.size();
      log.infof("session-limiter's configured realm session limit: %s", userRealmLimit);
      log.infof("session-limiter's configured client session limit: %s", userClientLimit);
      log.infof(
        "session-limiter's count of total user sessions for the entire realm (could be apps other than web apps): %s",
        userSessionCountForRealm);
      log.infof("session-limiter's count of total user sessions for this keycloak client: %s",
        userSessionCountForClient);

      // First check if the user has too many sessions in this realm
      if (exceedsLimit(userSessionCountForRealm, userRealmLimit)) {
        log.info("Too many session in this realm for the current user.");
        handleLimitExceeded(context, userSessionsForRealm);
      } // otherwise if the user is still allowed to create a new session in the realm, check if this applies for this specific client as well.
      else if (exceedsLimit(userSessionCountForClient, userClientLimit)) {
        log.info("Too many sessions related to the current client for this user.");
        handleLimitExceeded(context, userSessionsForClient);
      } else {
        context.success();
      }
    } else {
      context.success();
    }
  }

  private void handleLimitExceeded(AuthenticationFlowContext context,
    List<UserSessionModel> userSessions) {
    switch (behavior) {
      case UserSessionLimitsAuthenticatorFactory.DENY_NEW_SESSION:
        log.info("Denying new session");
        context.failure(AuthenticationFlowError.INVALID_CLIENT_SESSION);
        break;
      case UserSessionLimitsAuthenticatorFactory.TERMINATE_OLDEST_SESSION:
        log.info("Terminating oldest session");
        logoutOldestSession(userSessions);
        context.success();
        break;
      default:
        break;
    }
  }

  private void logoutOldestSession(List<UserSessionModel> userSessions) {
    log.info("Logging out oldest session");
    Optional<UserSessionModel> oldest = userSessions.stream()
      .sorted(Comparator.comparingInt(UserSessionModel::getStarted)).findFirst();
    oldest.ifPresent(
      userSession -> AuthenticationManager.backchannelLogout(session, userSession, true));
  }
}

```

## 使用

authentication flow
![](20200608132937418_908857975.png)
点`config` 进行配置。
![](20200608133708658_234115439.png)
可设置 每个用户可以最多有几个session。对每个client最多可以有几个session。 如果session数量超过限制如何处理：有两种选择，一、 拒绝创建新session。二、结束老session。

已测可用。 有问题请留言。