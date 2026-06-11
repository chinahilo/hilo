<template>
  <view class="app-shell" :style="shellStyle">
    <!--
      轨魅网主 WebView：
      - 默认加载 https://www.guimei8.com/?from=app
      - 不在 App 端强行拦截 PDF / Excel / export / print 等关键词，避免误伤网页功能
      - 真正的外部网盘链接、真实文件链接，由下面的 JS 逻辑单独处理
    -->
    <web-view
      id="guimeiWebview"
      ref="guimeiWebview"
      class="main-webview"
      :src="webUrl"
      :style="webViewStyle"
      @load="onWebViewLoad"
      @error="onWebViewError"
      @message="onWebViewMessage"
    ></web-view>

    <!--
      兜底按钮：当 WebView 内 PDF 导出、Excel 导出、微信二维码登录等体验不佳时，
      用户可以把“当前正在访问的网页”交给系统浏览器处理。
    -->
    <view class="browser-open-button" :style="browserButtonStyle" @tap="openCurrentPageInBrowser">
      浏览器打开
    </view>
  </view>
</template>

<script>
// 轨魅网 App 第一版：增强版 WebView 壳
// 设计原则：稳定优先，不用 pdf / excel / export / download / print 等宽泛关键词拦截。

const HOME_URL = 'https://www.guimei8.com/?from=app'
const APP_NAME = '轨魅网'
const EXIT_INTERVAL = 2000

// 只有这些明确的真实文件后缀，才认为是“实际文件地址”。
const REAL_FILE_EXTENSIONS = ['pdf', 'xls', 'xlsx', 'csv', 'doc', 'docx', 'zip', 'rar', '7z']

// 第一版只把百度网盘和 123 网盘交给系统浏览器 / 对应 App。
const EXTERNAL_HOSTS = ['pan.baidu.com', '123pan.com', 'www.123pan.com']

export default {
  data() {
    return {
      webUrl: HOME_URL,
      currentUrl: HOME_URL,
      statusBarHeight: 0,
      safeAreaBottom: 0,
      lastBackPressTime: 0,
      pageWebview: null,
      innerWebview: null
    }
  },

  computed: {
    shellStyle() {
      return {
        backgroundColor: '#ffffff'
      }
    },

    webViewStyle() {
      // #ifdef APP-PLUS
      // App 端由 native webview.setStyle 再做一次精确控制；这里保留样式便于 H5 / 预览。
      return {
        top: `${this.statusBarHeight}px`,
        height: `calc(100vh - ${this.statusBarHeight}px)`,
        width: '100%'
      }
      // #endif

      // #ifndef APP-PLUS
      return {
        height: '100vh',
        width: '100%'
      }
      // #endif
    },

    browserButtonStyle() {
      const bottom = 20 + this.safeAreaBottom
      return {
        bottom: `${bottom}px`
      }
    }
  },

  onLoad() {
    this.initSystemUi()
  },

  onReady() {
    // web-view 是原生组件，需要页面渲染后再获取内部 WebView 实例。
    // 延迟一点可以避免部分安卓机型 children()[0] 暂时取不到的问题。
    setTimeout(() => {
      this.initNativeWebview()
    }, 400)
  },

  onShow() {
    // 故意不刷新页面：从后台、锁屏、切 App 回来时，保持用户原来的阅读 / 表单 / 计算状态。
  },

  onHide() {
    // 故意不销毁、不 reload，避免后台回来后文章、计算器、登录状态丢失。
  },

  onBackPress() {
    // 安卓返回键：网页能返回就先返回网页；不能返回时，两秒内二次返回才退出 App。
    // 返回 true 表示由本页面接管返回事件。
    this.handleAndroidBack()
    return true
  },

  methods: {
    initSystemUi() {
      const systemInfo = uni.getSystemInfoSync()
      this.statusBarHeight = systemInfo.statusBarHeight || 0

      const safeAreaBottom = systemInfo.safeAreaInsets && systemInfo.safeAreaInsets.bottom
      this.safeAreaBottom = safeAreaBottom || 0

      // #ifdef APP-PLUS
      if (typeof plus !== 'undefined') {
        plus.navigator.setStatusBarBackground('#FFFFFF')
        plus.navigator.setStatusBarStyle('dark')
      }
      // #endif
    },

    initNativeWebview() {
      // #ifdef APP-PLUS
      if (typeof plus === 'undefined' || !this.$scope || !this.$scope.$getAppWebview) {
        return
      }

      this.pageWebview = this.$scope.$getAppWebview()
      const children = this.pageWebview.children()
      this.innerWebview = children && children.length ? children[0] : null

      if (!this.innerWebview) {
        return
      }

      // 让网页从状态栏下方开始显示，避免压住系统时间、电量、信号栏。
      this.innerWebview.setStyle({
        top: `${this.statusBarHeight}px`,
        bottom: '0px'
      })

      this.setupNarrowUrlInterception()
      this.injectLinkClickHelper()
      // #endif
    },

    setupNarrowUrlInterception() {
      // #ifdef APP-PLUS
      if (!this.innerWebview || !this.innerWebview.overrideUrlLoading) {
        return
      }

      // 只拦截两类明确地址：
      // 1. 百度网盘 / 123 网盘；
      // 2. URL 路径明确以真实文件后缀结尾。
      // 不拦截 pdf、excel、export、print 等普通关键词。
      const fileExtPattern = REAL_FILE_EXTENSIONS.join('|')
      const urlPattern = `^https?://(([^/]+\\.)?(pan\\.baidu\\.com)|123pan\\.com|www\\.123pan\\.com)(/|$)|^https?://[^?#]+\\.(${fileExtPattern})(\\?|#|$)`

      this.innerWebview.overrideUrlLoading(
        {
          mode: 'reject',
          match: urlPattern
        },
        event => {
          const url = event && event.url ? event.url : ''
          this.handleSpecialUrl(url)
        }
      )
      // #endif
    },

    injectLinkClickHelper() {
      // #ifdef APP-PLUS
      if (!this.innerWebview || !this.innerWebview.evalJS) {
        return
      }

      // 注入轻量 JS：只辅助捕获用户点击到的链接，并把当前 URL 回传给 App。
      // 这不是宽泛下载拦截；真正是否处理，仍由 App 端的 isExternalUrl / isRealFileUrl 严格判断。
      const script = `
        (function () {
          if (window.__GUIMEI_APP_HELPER_INSTALLED__) return;
          window.__GUIMEI_APP_HELPER_INSTALLED__ = true;

          function postToApp(payload) {
            try {
              if (window.uni && window.uni.postMessage) {
                window.uni.postMessage({ data: payload });
              }
            } catch (e) {}
          }

          document.addEventListener('click', function (event) {
            var el = event.target;
            while (el && el.tagName !== 'A') {
              el = el.parentNode;
            }
            if (!el || !el.href) return;
            postToApp({ type: 'link-click', url: el.href, pageUrl: location.href });
          }, true);

          window.addEventListener('hashchange', function () {
            postToApp({ type: 'page-url', pageUrl: location.href });
          });
        })();
      `

      this.innerWebview.evalJS(script)
      // #endif
    },

    onWebViewLoad(event) {
      const url = event && event.detail && event.detail.src ? event.detail.src : ''
      if (url) {
        this.currentUrl = url
      }

      // 每次页面加载后补一次注入，保证普通页面跳转后仍能捕获外部链接点击。
      setTimeout(() => {
        this.injectLinkClickHelper()
      }, 300)
    },

    onWebViewError() {
      uni.showToast({
        title: '网页加载失败，请检查网络',
        icon: 'none'
      })
    },

    onWebViewMessage(event) {
      const messages = event && event.detail && event.detail.data ? event.detail.data : []
      const list = Array.isArray(messages) ? messages : [messages]

      list.forEach(message => {
        if (!message) return

        if (message.pageUrl) {
          this.currentUrl = message.pageUrl
        }

        if (message.type === 'link-click' && message.url && this.shouldHandleSpecialUrl(message.url)) {
          this.handleSpecialUrl(message.url)
        }
      })
    },

    shouldHandleSpecialUrl(url) {
      return this.isExternalUrl(url) || this.isRealFileUrl(url)
    },

    handleSpecialUrl(url) {
      if (!url || this.isUnsupportedVirtualUrl(url)) {
        return
      }

      if (this.isExternalUrl(url)) {
        this.openUrlOutsideApp(url)
        return
      }

      if (this.isRealFileUrl(url)) {
        this.downloadAndOpenFile(url)
      }
    },

    isExternalUrl(url) {
      try {
        const parsed = new URL(url)
        const hostname = parsed.hostname.toLowerCase()
        return EXTERNAL_HOSTS.includes(hostname)
      } catch (error) {
        return false
      }
    },

    isRealFileUrl(url) {
      if (!url || this.isUnsupportedVirtualUrl(url)) {
        return false
      }

      try {
        const parsed = new URL(url)
        const pathname = decodeURIComponent(parsed.pathname || '').toLowerCase()
        return REAL_FILE_EXTENSIONS.some(ext => pathname.endsWith(`.${ext}`))
      } catch (error) {
        return false
      }
    },

    isUnsupportedVirtualUrl(url) {
      const lowerUrl = String(url).trim().toLowerCase()
      return (
        !lowerUrl ||
        lowerUrl.startsWith('blob:') ||
        lowerUrl.startsWith('javascript:') ||
        lowerUrl === 'about:blank' ||
        lowerUrl.startsWith('data:')
      )
    },

    openCurrentPageInBrowser() {
      const url = this.getBestCurrentUrl()
      if (!url || this.isUnsupportedVirtualUrl(url)) {
        uni.showToast({
          title: '当前页面暂不能用浏览器打开',
          icon: 'none'
        })
        return
      }

      this.openUrlOutsideApp(url)
    },

    getBestCurrentUrl() {
      // #ifdef APP-PLUS
      if (this.innerWebview && this.innerWebview.getURL) {
        const nativeUrl = this.innerWebview.getURL()
        if (nativeUrl) return nativeUrl
      }
      // #endif

      return this.currentUrl || HOME_URL
    },

    openUrlOutsideApp(url) {
      // #ifdef APP-PLUS
      if (typeof plus !== 'undefined') {
        plus.runtime.openURL(
          url,
          () => {
            uni.showToast({
              title: '无法打开外部浏览器',
              icon: 'none'
            })
          }
        )
        return
      }
      // #endif

      // H5 预览兜底。
      window.open(url, '_blank')
    },

    downloadAndOpenFile(url) {
      // #ifndef APP-PLUS
      this.openUrlOutsideApp(url)
      return
      // #endif

      // #ifdef APP-PLUS
      uni.showLoading({
        title: '正在下载文件'
      })

      uni.downloadFile({
        url,
        success: result => {
          uni.hideLoading()

          if (result.statusCode !== 200 || !result.tempFilePath) {
            uni.showToast({
              title: '文件下载失败，请用浏览器打开',
              icon: 'none'
            })
            return
          }

          uni.showModal({
            title: '下载完成',
            content: '是否立即打开文件？',
            confirmText: '打开',
            cancelText: '取消',
            success: modalResult => {
              if (modalResult.confirm) {
                this.openLocalFile(result.tempFilePath)
              }
            }
          })
        },
        fail: () => {
          uni.hideLoading()
          uni.showModal({
            title: '下载失败',
            content: '当前文件无法在 App 内下载，是否改用系统浏览器打开？',
            confirmText: '浏览器打开',
            cancelText: '取消',
            success: modalResult => {
              if (modalResult.confirm) {
                this.openUrlOutsideApp(url)
              }
            }
          })
        }
      })
      // #endif
    },

    openLocalFile(filePath) {
      // #ifdef APP-PLUS
      if (typeof plus !== 'undefined') {
        plus.runtime.openFile(
          filePath,
          {},
          () => {
            uni.showToast({
              title: '打开失败，请安装 WPS、PDF 阅读器等软件',
              icon: 'none',
              duration: 2500
            })
          }
        )
      }
      // #endif
    },

    handleAndroidBack() {
      // #ifdef APP-PLUS
      if (this.innerWebview && this.innerWebview.canBack) {
        this.innerWebview.canBack(event => {
          if (event.canBack) {
            this.innerWebview.back()
          } else {
            this.confirmExitApp()
          }
        })
        return
      }
      // #endif

      this.confirmExitApp()
    },

    confirmExitApp() {
      const now = Date.now()
      if (now - this.lastBackPressTime < EXIT_INTERVAL) {
        // #ifdef APP-PLUS
        if (typeof plus !== 'undefined') {
          plus.runtime.quit()
          return
        }
        // #endif
      }

      this.lastBackPressTime = now
      uni.showToast({
        title: `再按一次退出${APP_NAME}`,
        icon: 'none',
        duration: EXIT_INTERVAL
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.app-shell {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.main-webview {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
}

.browser-open-button {
  position: fixed;
  right: 16px;
  z-index: 9999;
  padding: 8px 12px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.94);
  color: #333333;
  font-size: 13px;
  line-height: 18px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.18);
  border: 1px solid rgba(0, 0, 0, 0.08);
}

.browser-open-button:active {
  opacity: 0.75;
}
</style>
