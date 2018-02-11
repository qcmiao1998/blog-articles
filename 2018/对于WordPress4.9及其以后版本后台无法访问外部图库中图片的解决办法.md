# 对于 WordPress 4.9 及其以后版本后台无法访问外部图库中图片的解决办法

## 概述

在更新到2017年11月16日发布的 `WordPress 4.9` 后，如果你使用了第三方图床（或是基于云储存的外置图片库）并开启了防盗链， 那么当你在后台媒体库查看图片或在编辑器中编辑文章时将无法成功预览图片。使用浏览器调试功能发现对图片的请求返回了 `403` ，对图片的正常访问被防盗链用的 Referer 白名单阻拦在外了。

## 原因

通过查看浏览器发出的 HTTP 请求，可以发现，文章中的访问的 [Referrer Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy) 是浏览器默认的 `no-referrer-when-downgrade` ，而管理员面板中的请求则以 `same-origin` 发出。故使用外部图库时，由于对图片的访问属于跨域操作，请求并未附带 referrer 信息，当图库的防盗链设置为不允许 referrer 为空时就会发生拒绝访问，图片无法显示的情况。

通过检索 WordPress 的源代码，可以发现该情况是 `4.9` 版本后新加入的 `wp_admin_headers()` 方法造成的。该方法当前（`WordPress 4.9.4` 中）位于 `wp-admin/includes/misc.php` 文件的第 `1148` 行，内容如下：

```PHP
/**
 * Send a referrer policy header so referrers are not sent externally from administration screens.
 *
 * @since 4.9.0
 */
function wp_admin_headers() {
	$policy = 'same-origin';

	/**
	 * Filters the admin referrer policy header value. Default 'same-origin'.
	 *
	 * @since 4.9.0
	 * @link https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy
	 *
	 * @param string $policy The referrer policy header value.
	 */
	$policy = apply_filters( 'admin_referrer_policy', $policy );

	header( sprintf( 'Referrer-Policy: %s', $policy ) );
}
```

同时，在 `wp-admin/includes/admin-filters.php` 中添加了如下的两个动作。

```PHP
add_action( 'admin_init', 'wp_admin_headers' );
add_action( 'login_init', 'wp_admin_headers' );
```

正是这两个动作使属于管理员面板的页面在加载时会向浏览器发送 `Referrer-Policy: same-origin` 的头信息，使浏览器在请求跨域资源时不会添加 referrer 信息，造成前述的问题。

## 解决办法

在代码修改说明（[Changeset 41741](https://core.trac.wordpress.org/changeset/41741)）中指出，在后台使用 `same-origin` 的 referrer 策略是为了保障网站的安全。该策略可使跨站资源无法得到后台的具体地址，增强网站的安全性。

>This sets a referrer policy of same-origin which adds hardening by preventing a referrer being sent from the admin area or login screens to other origins. This helps prevent unwanted exposure of potentially sensitive information that may be contained within URLs.

同时代码修改说明中也给出了问题的解决方案，那就是移除 `admin-filters.php` 文件中的那两个动作。

>This change introduces a new filter, `admin_referrer_policy`, for filtering the referrer policy header value. The header can be disabled if necessary by removing the `wp_admin_headers` action from the `admin_init` and `login_init` hooks.

但考虑到后台使用 `no-referrer-when-downgrade` 之类的低安全性 referrer 策略着实不利于网站安全，我选择了修改 `wp_admin_headers()` 方法，将其中的 `same-origin` 改为了 `strict-origin-when-cross-origin`。该策略在同域时会正常传输 referrer，在跨域时将仅传输来源网站的域名，同时若出现 HTTPS->HTTP 的不安全情形，referrer 的传递也会终止。

>**same-origin**  
> * A referrer will be sent for same-site origins, but cross-origin requests will contain no referrer information.

>**strict-origin-when-cross-origin**  
> * Send a full URL when performing a same-origin request, only send the origin of the document to a-priori as-much-secure destination (HTTPS->HTTPS), and send no header to a less secure destination (HTTPS->HTTP).

## 总结

如果你的网站并不涉及跨域媒体文件调用时的防盗链问题，上述的修改是不需要的，`same-origin` 的策略必然是更加安全的。但倘若遇到了和我一样的情况，并且不希望牺牲图片的安全性，则可以参考上文中的方法进行文件的修改。但需要注意的是，修改网站的核心文件是很危险的行为，需要谨慎进行，否则极有可能造成网站的崩溃。在任何修改之前都请备份你的原始文件。同时，WordPress 版本的升级将导致修改的覆盖，为便于再次修改可在主机上建立一个小的 Shell 脚本来快速修改，参考如下：

```bash
cp ./html/wp-admin/includes/misc.php ./html/wp-admin/includes/misc-bak.php
sed -i "s/same-origin/strict-origin-when-cross-origin/g" ./html/wp-admin/includes/misc.php
```

如有需要，请根据实际情况自行修改上述内容。

## Reference
1. https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy
2. https://core.trac.wordpress.org/changeset/41741
3. https://core.trac.wordpress.org/ticket/42036
