<script>
export default {
  onLaunch() {
    // App 启动入口。第一版不做自动刷新、不做强制登录、不写假的第三方登录逻辑。
  },
  onShow() {
    // 从后台回到前台时保持 WebView 当前状态。
  },
  onHide() {
    // 进入后台时不主动销毁页面。
  }
}
</script>

<style lang="scss">
page {
  background-color: #ffffff;
}
</style>
