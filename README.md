## 微信jssdk ##
```
/**
 * @file 微信JS-SDK封装
 */

import apis from '../common/apis'
import query from '../common/query'
import axios from '../plugins/axios'
import loadScript from '../utils/load-script'
import { isWeChat, isAndroidWeChat, isIOSWeChat, encodeSearchParams } from './'
import track from 'common/sa'

const location = global.location
const REGEXP_SUCCESS = /^\w+:ok$/i

/**
 * 获取当前页面URL（去除hash）
 * @returns {string} 页面URL
 */
function getCurrentURL() {
  return location.href.split('#')[0]
}

export default {
  jWeixin: null,
  promise: null,
  signature: null,
  url: '',

  /**
   * 微信JS-SDK初始化
   * @returns {Promise} Promise实例
   */
  init() {
    if (!this.promise) {
      this.promise = new Promise((resolve, reject) => {
        if (!isWeChat()) {
          reject('请在微信中打开当前页面')
          return
        }

        Promise.all([
          this.initScript(),
          this.initSignature(),
        ]).then(([jWeixin, signature]) => {
          jWeixin.config({
            appId: signature.appId,
            timestamp: signature.timestamp,
            nonceStr: signature.nonceStr,
            signature: signature.signature,
            jsApiList: [
              'showMenuItems',
              'hideMenuItems',
              'onMenuShareTimeline',
              'onMenuShareAppMessage',
              'onMenuShareQQ',
              'onMenuShareWeibo',
              'onMenuShareQZone',
              'chooseWXPay',
              'addCard',
              'closeWindow',
            ],
          })

          jWeixin.ready(() => {
            this.promise = null
            resolve(jWeixin)
          })

          jWeixin.error(res => reject(res.errMsg))
        }).catch((e) => {
          this.promise = null
          reject(e)
        })
      })
    }

    // 确保同时多次调用时，只进行一次初始化
    return this.promise
  },

  /**
   * 加载脚本文件
   * @returns {Promise} Promise实例
   */
  initScript() {
    return new Promise((resolve, reject) => {
      if (this.jWeixin) {
        resolve(this.jWeixin)
        return
      }

      loadScript('//res.wx.qq.com/open/js/jweixin-1.2.0.js', {
        async: true,
        defer: true,
      }).then(() => {
        resolve(this.jWeixin = (global.jWeixin || global.wx))
      }).catch(() => {
        reject('加载微信JSSDK文件失败')
      })
    })
  },

  /**
   * 获取签名信息
   * @returns {Promise} Promise实例
   */
  initSignature() {
    return new Promise((resolve, reject) => {
      const url = getCurrentURL()

      if (this.signature && url === this.url) {
        resolve(this.signature)
        return
      }

      this.url = url

      axios.request({
        url: apis.common.wechat.sign,
        method: 'POST',
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
        },

        // 此接口必须使用URLSearchParams传参
        data: encodeSearchParams({
          url,

          // 如果有券ID，则后端以该券ID对应的商户公众号进行签名，否则使用默认的X卡公众号进行签名
          channelId: query.channelId,
        }),
      }).then((data) => {
        const signature = data.result

        if (!signature) {
          reject('签名信息无效')
          return
        }

        // 判断路由是否已经变化
        if (signature.url !== getCurrentURL()) {
          reject('签名URL与当前页面URL不一致')
          return
        }

        this.signature = signature
        resolve(signature)
      }).catch(() => {
        reject('获取签名信息失败')
      })
    })
  },

  /**
   * init的简化版
   * @param {Function} resolve - 初始化成功回调函数
   * @param {Function} reject - 初始化失败回调函数
   * @returns {Promise} Promise实例
   */
  ready(resolve, reject) {
    return this.init().then(resolve).catch(reject)
  },

  /**
   * 分享微信分享成功时的数据埋点
   * @param {Object} options - 需要sa埋点的数据内容
   * @param {String} mode - 当前分享模式
   * @returns {Object} 分享参数
   */
  getShareTrack(options, mode) {
    if (!options.trackData) {
      return options
    }
    return Object.assign({}, options, {
      // 微信分享回调
      success() {
        const param = Object.assign({}, options.trackData, {
          shareMethod: mode,
        })
        track.share(param)
      },
    })
  },
  /**
   * 微信分享接口封装
   * @param {Object} options - 配置项
   * @returns {Promise} Promise实例
   */
  share(options) {
    if (options.imgUrl && options.imgUrl.startsWith('//')) {
      options.imgUrl = `http:${options.imgUrl}`
    }
    return new Promise((resolve, reject) => {
      options = Object.assign({
        // XXX: iOS版微信自动配置的链接不会随着路由切换自动更新，而location.href会正常更新
        link: location.href,
      }, options)

      // XXX: 微信不支持//开头的图片地址
      if (options.imgUrl && String(options.imgUrl).indexOf('//') === 0) {
        options.imgUrl = `http:${options.imgUrl}`
      }

      this.ready((jWeixin) => {
         // 分享到朋友圈
        jWeixin.onMenuShareTimeline(this.getShareTrack(options, '朋友圈'))

        // 分享给朋友
        jWeixin.onMenuShareAppMessage(this.getShareTrack(options, '分享给朋友'))

        // 分享到QQ
        jWeixin.onMenuShareQQ(this.getShareTrack(options, '分享到QQ'))

        // 分享到腾讯微博
        jWeixin.onMenuShareWeibo(this.getShareTrack(options, '腾讯微博'))

        // 分享到QQ空间
        jWeixin.onMenuShareQZone(this.getShareTrack(options, 'QQ空间'))

        resolve()
      }, reject)
    })
  },

  /**
   * 批量显示功能按钮接口
   * @param {Object} options - 配置项
   * @returns {Promise} Promise实例
   */
  show(options) {
    return new Promise((resolve, reject) => {
      this.ready((jWeixin) => {
        jWeixin.showMenuItems(Object.assign({}, options, {
          /**
           * 调用成功回调函数
           * @param {Object} response - 响应数据
           */
          success(response) {
            if (REGEXP_SUCCESS.test(response.errMsg)) {
              resolve()
            } else {
              reject()
            }
          },

          /**
           * 调用失败回调函数
           */
          fail: () => {
            reject()
          },
        }))
      }, reject)
    })
  },

  /**
   * 批量隐藏功能按钮接口
   * @param {Object} options - 配置项
   * @returns {Promise} Promise实例
   */
  hide(options) {
    return new Promise((resolve, reject) => {
      this.ready((jWeixin) => {
        jWeixin.hideMenuItems(Object.assign({}, options, {
          /**
           * 调用成功回调函数
           * @param {Object} response - 响应数据
           */
          success(response) {
            if (REGEXP_SUCCESS.test(response.errMsg)) {
              resolve()
            } else {
              reject()
            }
          },

          /**
           * 调用失败回调函数
           */
          fail: () => {
            reject()
          },
        }))
      }, reject)
    })
  },

  /**
   * 批量添加卡券接口封装
   * @param {Object} options - 配置项
   * @returns {Promise} Promise实例
   */
  addCard(options) {
    return new Promise((resolve, reject) => {
      this.ready((jWeixin) => {
        jWeixin.addCard(Object.assign({}, options, {
          /**
           * 调用成功回调函数
           * @param {Object} response - 响应数据
           */
          success(response) {
            if (REGEXP_SUCCESS.test(response.errMsg)) {
              resolve()
            } else {
              reject()
            }
          },

          /**
           * 调用失败回调函数
           */
          fail: () => {
            reject()
          },
        }))
      }, reject)
    })
  },

  /**
   * 微信支付接口封装
   * @param {Object} options - 配置项
   * @returns {Promise} Promise实例
   */
  pay(options) {
    return new Promise((resolve, reject) => {
      if (!isAndroidWeChat() && !isIOSWeChat()) {
        reject('请在手机微信中打开当前页面进行支付')
        return
      }

      const WeixinJSBridge = global.WeixinJSBridge

      /**
       * 微信Bridge回调函数
       */
      function onBridgeReady() {
        const params = Object.assign({
          package: options.wxPackage,
        }, options)

        WeixinJSBridge.invoke('getBrandWCPayRequest', params, (response) => {
          // response schema: { err_code: String, err_desc: String, err_msg: String }
          switch (response.err_msg) {
            case 'get_brand_wcpay_request:ok':
              resolve()
              break

            case 'get_brand_wcpay_request:cancel':
              reject(response.err_desc || '支付取消')
              break

            // case 'get_brand_wcpay_request:fail':
            default:
              reject(response.err_desc || '支付失败')
          }
        })
      }

      if (typeof WeixinJSBridge === 'undefined') {
        if (document.addEventListener) {
          document.addEventListener('WeixinJSBridgeReady', onBridgeReady, false)
        } else if (document.attachEvent) {
          document.attachEvent('WeixinJSBridgeReady', onBridgeReady)
          document.attachEvent('onWeixinJSBridgeReady', onBridgeReady)
        }
      } else {
        onBridgeReady()
      }
    })
  },
}
```
