---
layout: post
title: 使用Python发送电子邮件
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
使用python发送邮件并不难，这里使用的是SMTP协议。
  Python标准库中内置了smtplib，使用它发送邮件只需提供邮件内容与发送者的凭证即可。
  代码如下：
  

```Python
# coding:utf-8
import smtplib
from email.mime.text import MIMEText
import time
import os
import sys

def send_mail(subject, body, mail_to, username, password, mail_type='plain'):
    assert isinstance(mail_to, list) == True
    msg = MIMEText(body, _subtype=mail_type)
    # 定义标题
    msg['Subject'] = subject
    # 定义发信人
    msg['From'] = username
    # 
    msg['To'] = ';'.join(mail_to)
    # 定义发送时间（不定义的可能有的邮件客户端会不显示发送时间）
    msg['date'] = time.strftime('%a, %d %b %Y %H:%M:%S %z')
    try:
        smtp = smtplib.SMTP()
        # 连接SMTP服务器，
        smtp.connect('smtp.exmail.qq.com')
        # 用户名密码
        smtp.login(username, password)
        smtp.sendmail(username, mail_to, msg.as_string())
        smtp.quit()
        return True
    except Exception as e:
        print "send mail error:%s"%e
        return False

if __name__ == "__main__":

    if len(sys.argv) < 2:
        print 'Usage : python mail.py object_mail'
        sys.exit()

    subject = '你好，这是一封测试邮件'
    body = '''
这是经过我们的python程序发送的一封邮件，请勿直接回复
    '''
    mail_to = [sys.argv[1]]
    username = 'inevermore@foxmail.com'
    password = os.getenv('PASSWORD')
    send_mail(subject, body, mail_to, username, password)
```
		

这里有几点：



  1.我把邮箱的密码设置在了环境变量中，可以使用 export PASSWORD=’XXXXXXXX’ 设置




  2.发送者的邮箱必须开通SMTP发送服务，否则程序会抛出异常。




  3.使用 python mail.py 收件人的地址 即可发送成功 

			