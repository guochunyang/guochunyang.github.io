---
layout: post
title: 微信支付的开发流程
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
最近在公司做了微信支付的接入，这里总结下开发的一些经验
  注意，我使用的是微信开放平台的支付，与手机app相关，而与公众账号无关。
   
  
###微信支付的主要操作流程
     1.用户浏览app，选定商品然后下单。
    2.服务器处理订单逻辑，开始正式发起支付流程
    3.首先，后台服务器向weixin服务器发起请求，获取一个token。
    4.后台服务器拿到token，使用和其他参数加密，再次向weixin服务器发起请求，获取一个预支付prepayid
    5.后台服务器将该prepayid返回给app客户端
    6.app调用手机上的微信控件，完成付款流程。
    7.app向后台服务器发起一个回调请求，通知服务器交易完成。
    8.weixin服务器处理完所有的流程后，向后台服务器发起一个post请求，正式通知后台服务器交易完毕
    
  
###上面流程的一些注意点
     1.每次获取的token是有时效的，默认是7200s，而且每天最多获取200次，因此最好放到redis中缓存起来，等失效后再去重新获取
    2.app发起的回调默认是不可靠的，后台应该尽可能（不是必须）向微信服务器发起订单查询，查询本次交易的结果。
    3.weixin服务器向后台发起的notify，才是确保交易完成的最后屏障。后台服务器确认后必须返回“success”，否则weixin服务器会尝试重发请求。
    
  
###获取token
  这步很简单，发送一个get请求即可。只需配置正确参数。
  

```Python
'''从微信服务器获取token'''
    def _getAccessTokenFromWeixin(self):
        response = requests.get(self.tokenUrl % (self.appId, self.appSecret))
        if response.status_code == 200:
            text = response.text
            tokenInfo = json.loads(text)
            try:
                token = tokenInfo['access_token']
                expires_in = tokenInfo['expires_in']
                self._writeWeixinTokenLog(token, self.order_no)
                return token
            except KeyError:
                return None #token获取失败
        return None #http请求失败
```
		

 



###获取prepayid


在微信支付的开发流程中，最繁琐的就是获取prepayid。


这一步我们需要组装这样一个参数：




```C++
{
"appid":"wxd930ea5d5a258f4f",
"traceid":"test_1399514976",
"noncestr":"e7d161ac8d8a76529d39d9f5b4249ccb ",
"timestamp":1399514976, "package":"bank_type=WX&body=%E6%94%AF%E4%BB%98%E6%B5%8B%E8%AF%
95&fee_type=1&input_charset=UTF-8&notify_url=http%3A%2F%2Fweixin.qq.com&out_trade_ no=7240b65810859cbf2a8d9f76a638c0a3&partner=1900000109&spbill_create_ip=196.168.1.1& total_fee=1&sign=7F77B507B755B3262884291517E380F8",
"sign_method":"sha1", "app_signature":"7f77b507b755b3262884291517e380f8"
}
```
		

 



###组装package


这里的第一步就是组装package：




```C++
"package":"bank_type=WX&body=%E6%94%AF%E4%BB%98%E6%B5%8B%E8%AF%
95&fee_type=1&input_charset=UTF-8&notify_url=http%3A%2F%2Fweixin.qq.com&out_trade_ no=7240b65810859cbf2a8d9f76a638c0a3&partner=1900000109&spbill_create_ip=196.168.1.1& total_fee=1&sign=7F77B507B755B3262884291517E380F8",
```
		

组装package需要的参数如上面代码所示，所以我们需要准备一个params，然后准备签名，签名流程如下：


1.按照key的字典序，对params进行排序，然后拼接成字符串，注意这些key不包括sign


2.在上面的字符串后面拼接key=paternerKey，然后对整个字符串进行md5签名，然后转换成大写，此时我们就得到了签名


然后我们将所有params的value进行urlencode转码，然后后面拼接上sign=signValue，就得到了package字符串。


这里创建MD5的过如下：




```Python
def createMD5Signature(self, signParams):
        '''先排序'''
        sortedParams = sorted(signParams.iteritems(), key=lambda d:d[0])
        '''拼接'''   
        stringSignTemp = "&".join(["%s=%s" % (item[0], item[1]) for item in sortedParams if item[0] != 'sign' and '' != item[1]])
        #加上财付通商户权限密钥
        stringSignTemp += '&key=%s' % (self.partnerKey)
        #使用MD5进行签名，然后转化为大写
        stringSign = hashlib.md5(stringSignTemp).hexdigest().upper()    #Upper
        return stringSign
```
		

组装package的代码：




```Python
def getPackage(self, packageParams):
        '''先获取params的sign，然后将params进行urlencode，最后拼接，加上sign'''
        sign = self.createMD5Signature(packageParams)
        packageParams = sorted(packageParams.iteritems(), key=lambda d:d[0])
        stringParams = "&".join(["%s=%s" % (item[0], urllib.quote(str(item[1]))) for item in packageParams])
        stringParams += '&sign=%s' % (sign)
        return stringParams
```
		


###继续组装参数


得到package后，我们继续组装参数：


这里需要的参数为：




```C++
appid=wxd930ea5d5a258f4f
appkey=L8LrMqqeGRxST5reouB0K66CaY A WpqhA Vsq7ggKkxHCOastWksvuX1uvmvQcl xaHoYd3ElNBrNO2DHnnzgfVG9Qs473M3DTOZug5er46FhuGofumV8H2FVR9qkjSlC5K
noncestr=e7d161ac8d8a76529d39d9f5b4249ccb
package=bank_type=WX&body=%E6%94%AF%E4%BB%98%E6%B5%8B%E8%AF%95 &fee_type=1&input_charset=UTF-8&notify_url=http%3A%2F%2Fweixin.qq.com&out_trade_no =7240b65810859cbf2a8d9f76a638c0a3&partner=1900000109&spbill_create_ip=196.168.1.1&tot al_fee=1&sign=7F77B507B755B3262884291517E380F8
timestamp=1399514976 </pre>

  <pre>traceid=test_1399514976
```
		

注意这里有个坑：


参与签名的是上面的参数，`但是最后的参数中不包括appKey，签名后要记得删除`。


1.所有参数按照字典序排序，然后拼接


2.进行sha1签名，拼接到上面字符串的后面


3.注意这里要删除appKey，然后加上sign


获取sha1签名的代码如下：




```Python
def createSHA1Signature(self, params):
        '''先排序，然后拼接'''
        sortedParams = sorted(params.iteritems(), key=lambda d:d[0]) 
        stringSignTemp = "&".join(["%s=%s" % (item[0], item[1]) for item in sortedParams])
        stringSign = hashlib.sha1(stringSignTemp).hexdigest()
        return stringSign
```
		

随后我们获取到这样的参数：




```C++
{
"appid":"wxd930ea5d5a258f4f", </pre>

  <pre>"noncestr":"e7d161ac8d8a76529d39d9f5b4249ccb", </pre>

  <pre>"package":"Sign=WXpay";
"partnerid":"1900000109" </pre>

  <pre>"prepayid":"1101000000140429eb40476f8896f4c9", </pre>

  <pre>"sign":"7ffecb600d7157c5aa49810d2d8f28bc2811827b", </pre>

  <pre>"timestamp":"1399514976"
}
```
		

 



###获取prepayid


代码如下：




```Python
'''获取预支付prepayid'''
    def gerPrepayId(self, token, requestParams):
        '''将参数，包括package，进行json化，然后发起post请求'''
        data = json.dumps(requestParams)
        response = requests.post(self.gateUrl % (token), data=data)
        if response.status_code == 200:
            text = response.text
            text = json.loads(text)
            errcode = text['errcode']
            if errcode == 0:
                return text['prepayid']
        return None
```
		

我们获取的prepayid格式应该是这样：




```C++
{"prepayid":"1101000000140429eb40476f8896f4c9","errcode":0,"errmsg":"Success"}
```
		

 



###再次签名


这里采用上面sha1的签名方式再次签名，获取到下面的参数：




```C++
{
"appid":"wxd930ea5d5a258f4f", </pre>

  <pre>"noncestr":"e7d161ac8d8a76529d39d9f5b4249ccb", </pre>

  <pre>"package":"Sign=WXpay";
"partnerid":"1900000109" </pre>

  <pre>"prepayid":"1101000000140429eb40476f8896f4c9", </pre>

  <pre>"sign":"7ffecb600d7157c5aa49810d2d8f28bc2811827b", </pre>

  <pre>"timestamp":"1399514976"
}
```
		

后台服务器将该结果返回给app，此时app即可发起支付。


上面的流程代码为：




```Python
'''接收app的请求，返回prepayid'''
class WeixinRequirePrePaidHandler(BasicTemplateHandler):

    '''这个方法在OrdersAddHandler中被调用'''
    @staticmethod
    def getPrePaidResult(order_no, total_pay, product_name, client_ip):
        '''封装了常用的签名算法'''
        weixinRequestHandler = WeixinRequestHandler(order_no)
        '''收集订单相关信息'''
        addtion = str(random.randint(10, 100)) #产生一个两位的数字，拼接在订单号的后面
        out_trade_no = str(order_no) + addtion
        order_price = float(total_pay) #这里必须允许浮点数，后面转化成分之后转化为int
        #order_price = 0.01 #测试
        remote_addr = client_ip #客户端的IP地址
        print remote_addr
        current_time = int(time.time())
        order_create_time = str(current_time)
        order_deadline = str(current_time + 20*60)

        '''这里的一些参数供下面使用'''
        noncestr = hashlib.md5(str(random.random())).hexdigest()
        timestamp = str(int(time.time()))
        pack = 'Sign=WXPay'

        '''获取token'''
        access_token = weixinRequestHandler.getAccessToken()
        logging.info("get token: %s" % access_token)
        if access_token:
            '''设置package参数'''
            packageParams = {}
            packageParams['bank_type'] = 'WX'   #支付类型
            packageParams['body'] = product_name  #商品名称
            packageParams['fee_type'] = '1'     #人民币 fen
            packageParams['input_charset'] = 'GBK' #GBK
            packageParams['notify_url'] = config['notify_url'] #post异步消息通知
            packageParams['out_trade_no'] = str(out_trade_no) #订单号
            packageParams['partner'] = config['partnerId'] #商户号
            packageParams['total_fee'] = str(int(order_price*100))   #订单金额,单位是分
            packageParams['spbill_create_ip'] = remote_addr #IP
            packageParams['time_start'] = order_create_time #订单生成时间
            packageParams['time_expire'] = order_deadline #订单失效时间

            '''获取package'''
            package = weixinRequestHandler.getPackage(packageParams)

            '''设置支付参数'''
            signParams = {}
            signParams['appid'] = config['appId']
            signParams['appkey'] = config['paySignKey'] #delete
            signParams['noncestr'] = noncestr
            signParams['package'] = package
            signParams['timestamp'] = timestamp
            signParams['traceid'] = 'mytraceid_001'

            '''生成支付签名'''
            app_signature = weixinRequestHandler.createSHA1Signature(signParams)
            '''增加不参与签名的额外参数'''
            signParams['sign_method'] = 'sha1'
            signParams['app_signature'] = app_signature

            '''剔除appKey'''
            del signParams['appkey']

            '''获取prepayid'''
            prepayid = weixinRequestHandler.gerPrepayId(access_token, signParams)
            if prepayid:

                '''使用拿到的prepayid再次准备签名'''
                pack = 'sign=WXPay'
                prepayParams = {}
                prepayParams['appid'] = config['appId']
                prepayParams['appkey'] = config['paySignKey']
                prepayParams['noncestr'] = noncestr
                prepayParams['package'] = pack
                prepayParams['partnerid'] = config['partnerId']
                prepayParams['prepayid'] = prepayid
                prepayParams['timestamp'] = timestamp

                '''生成签名'''
                sign = weixinRequestHandler.createSHA1Signature(prepayParams)

                '''准备输出参数'''
                returnParams = {}
                returnParams['status'] = 0
                returnParams['retmsg'] = 'success'
                returnParams['appid'] = config['appId']
                returnParams['noncestr'] = noncestr
                returnParams['package'] = pack
                returnParams['prepayid'] = prepayid
                returnParams['timestamp'] = timestamp
                returnParams['sign'] = sign
                returnParams['partnerId'] = config['partnerId']
                returnParams['addtion'] = addtion
                
            else:
                '''prepayid获取失败'''
                returnParams = {}
                returnParams['status'] = -1
                returnParams['retmsg'] = 'prepayid获取失败'
        else: 
            '''token获取失败'''
            returnParams = {}
            returnParams['status'] = -1
            returnParams['retmsg'] = 'token获取失败'

        '''生成json格式文本，然后返回给APP'''
        return returnParams
```
		


###后台异步通知


微信服务器发来的notify异步通知，才是支付成功的最终标志，这一步处于安全起见，我们必须进行延签：


延签代码如下：




```Python
def isTenpaySign(self, params):
        helper = WeixinRequestHandler()
        sign = helper.createMD5Signature(params)
        return params['sign'] == sign
```
		

整体流程如下：




```Python
'''微信服务器向后台发送的异步通知'''
class WeixinAppNotifyHandler(BasicTemplateHandler):
    def initialize(self):
        self.weixinResponseHandler = WeixinResponseHandler()

    def post(self):
        '''解析参数'''
        params = self.parseQueryString()

        '''验证是否是weixin服务器发回的消息'''
        verifyWeixinSign = self.weixinResponseHandler.isTenpaySign(params)
        '''处理订单'''
        if verifyWeixinSign:
            '''订单逻辑'''
            order_no = str(params['out_trade_no'])
            order_no = order_no[0:-2]
            print '%s paied successfully' % order_no
            self.saveWeixinReceipt(params)
            updateOrdersPaidByWeixin(order_no) #更新订单使用状态
            consumeCouponByOrderNo(order_no) #优惠券已经使用
            self.write("success")
        else:
            self.write("fail")

    def parseQueryString(self):
        '''获取url中所有的参数'''
        uri = self.request.uri
        '''解析出URI中的query字符串'''
        parseResult = urlparse.urlparse(uri) 
        query = parseResult.query
        '''解析query字符串'''
        params = urlparse.parse_qs(query)
        for item in params:
            params[item] = params[item][0].strip()
        return params
```
		

最后说明一点，用户在手机上付完款，并不算支付成功，`只有weixin服务器收到notify通知返回的success时，才算交易最终成功`，此时我们的手机可以收到微信官方发来的一条消息。

			