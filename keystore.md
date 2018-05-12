## 一张图看懂AndroidKeyStore原理

- ![](https://github.com/daBisNewBee/Notes/blob/master/pic/AndroidkeyStore.jpg)

- “AndroidKeyStore”的实质

  >  一个Native进程，名为“android.security.keystore”

  ```
  HWVKY:/system/lib $ ps|grep keystore
  ps|grep keystore
  keystore  527   1     19380  2408           0 0000000000 S /system/bin/keystore
  ```

- 获取”AndroidKeyStore“的两种方式

  - 通过注册Provider到JCE，可索引到”AndroidKeyStore“

    ```
    public class AndroidKeyStoreProvider extends Provider {
        public static final String PROVIDER_NAME = "AndroidKeyStore";

        public AndroidKeyStoreProvider() {
            super(PROVIDER_NAME, 1.0, "Android KeyStore security provider");

            // java.security.KeyStore
            put("KeyStore." + AndroidKeyStore.NAME, AndroidKeyStore.class.getName());

            // java.security.KeyPairGenerator
            put("KeyPairGenerator.RSA", AndroidKeyPairGenerator.class.getName());
        }
    }
    ```

  - 通过KeyChain

- 如何在Native空间，无法导出AndroidKeyStore中私钥材料的情况下，对TEE中的KeyStore进行私钥运算？

  > 通过OpenSSL的”Engine“方式，重载了私钥运算方法

  - 即”Android keystore engine“
  - 一个在native空间直接操作该Engine的[例子](https://github.com/daBisNewBee/NativeKeyStore.git)，核心实现：

  ```

  ENGINE_load_dynamic();

  // 获取到“keystore”Engine指针
  engine = ENGINE_by_id("keystore");

  // 增加引用计数
  ENGINE_init(engine));

  // 添加到全局链表
  ENGINE_add(engine));

  // 获取私钥
  EVP_PKEY *priKey = ENGINE_load_private_key(engine,"USRPKEY_orig_key_alias",NULL,NULL);

  // 获取公钥
  EVP_PKEY *pubKey = ENGINE_load_public_key(engine,"USRPKEY_orig_key_alias",NULL,NULL);

  ```

- Keychain与AndroidKeyStore同是对密钥库的操作，如何实现对权限的控制？

- Conscrypt 是什么？  与 Android的渊源

  Conscrypt ：一个Google提供的基于OpenSSL/BoringSSL的 JSP，实现了部分JCE和JSSE接口。近期完全使用了BoringSSL代替了OpenSSL，其实可看出，该JSP是BoringSSL在 Android上Java应用和OpenJDK的推广者。


> 各module对`Conscrypt `的使用情况：

|API|... | Android 5.0 | Android 5.1|Android 6.0|Android 7.0|...|Android P|
| --- | ---  | ---  | ---  | ---  | ---  |---  | ---  |
| KeyStore/KeyChain |  支持    | 支持 | 支持 | X | X | X | X |
| Provider（“AndroidOpenSSL”） | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 | X |

> 在Android 6.0及以后，KeyStore/KeyChain不在使用“Conscrypt ”访问密钥库，官方的说明是“可以增强认证手段”。参考[Android Keystore keys are no longer backed by Conscrypt.](https://gitlab.tubit.tu-berlin.de/justus.beyer/streamagame_platform_frameworks_base/commit/4a0ff7ca984d29bd34b02e54441957cad65e8b53)
>
> 在Android P及以后，索性就完全抛弃了 Conscrypt 这个JSP，由于与“BC”的接口重复，却不能带来益处。


#### TODO：

- Android 6.0及以后，Google 使用”BoringSSL“替换了”OpenSSL“，因此Engine方式获取密钥库的方法也被废除


#### 引用

- [Android Keystore keys are no longer backed by Conscrypt.](https://gitlab.tubit.tu-berlin.de/justus.beyer/streamagame_platform_frameworks_base/commit/4a0ff7ca984d29bd34b02e54441957cad65e8b53)
- [NativeKeyStore:一个在native空间操作密钥库的例子](https://github.com/daBisNewBee/NativeKeyStore.git)
- [Conscrypt 官方](https://github.com/google/conscrypt)
- [KeyStore介绍](https://github.com/doridori/Android-Security-Reference/blob/master/framework/keystore.md)
