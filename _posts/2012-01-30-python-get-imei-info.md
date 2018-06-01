---
layout: post
title: SL4A - 获取IMEI信息
description: 用python写了个获取手机IMEI信息的小脚本
tags: [Python, code]

comments: true
share: true
---
前段时间 [@红黄满](http://weibo.com/hysia) 同学用python写了个获取手机IMEI信息的[小脚本](http://pastebin.com/J6QK01HN)，今天小小的改动了一下，让它在Android上跑起来

<s>效果图因为放在SAE的博客，然后博客年代久矣被我删了，现在手头上也没Android设备，然后就没然后了，请看官老爷们自行脑补</s>

SL4A环境搭配([https://code.google.com/p/android-scripting/](https://code.google.com/p/android-scripting/))

<!-- more -->

{% highlight python %}
#!/usr/bin/python
# encoding:utf8

"""
IMEI 号码分析工具
update by hysia 

http://blog.hysia.com

http://weibo.com/hysia

2011/11/20
"""
import re
import urllib
import urllib2
import socket
import android

IMEI_INFO = {
    'Type Allocation Holder' : r'''Type Allocation Holder</td>[\s\S]*?<td class="basic-text">(.*?)</td>''',
    'Mobile Equipment Type' : r'''Mobile Equipment Type</td>[\s\S]*?<td class="basic-text">(.*?)</td>''',
    'GSM Implementation Phase' : r'''GSM Implementation Phase</td>[\s\S]*?<td class="basic-text">(.*?)</td>''',
    'Primary Market' : r'''Primary Market</td>[\s\S]*?<td class="basic-text">(.*?)</td>''',
    'Legal Basis for Allocation' : r'''Legal Basis for Allocation</td>[\s\S]*?<td class="basic-text">(.*?)</td>''',
    'Full IMEI Presentation' : r'''Full IMEI Presentation</td>[\s\S]*?<td class="basic-text">(.*?)</td>'''
}

def getUrlData(url,postData):
	retVal = None
	req = urllib2.Request(url,postData)
	f = urllib2.urlopen(req)
	try:
		retVal = f.read()
	except:
		print "Get Url Data Error!"
	finally:
		f.close

	return retVal

def analysisIMEI(num):
	retVal = {}
	url = "http://www.numberingplans.com/?page=analysis&sub=imeinr"
	postData = "i=%s" % num
	htmlData = getUrlData(url,postData)
	#print htmlData
	for info in IMEI_INFO:
		m = re.search(IMEI_INFO[info],htmlData)
		if m:
			retVal[info] = m.groups()[0].strip()
			#print retVal
	return retVal

if __name__ == '__main__':
	socket.setdefaulttimeout(20)
	droid = android.Android()

	try:
		imei = droid.getDeviceId()
		#print imei.result
		msg = u"检索IMEI信息..."
		droid.makeToast(msg)
		imeiInfo = analysisIMEI(imei.result)

		if imeiInfo:
			info = u"IMEI信息： %s \n" %imeiInfo['Full IMEI Presentation']
			info += u"手机制造商： %s \n" %imeiInfo['Type Allocation Holder']
			info += u"手机型号： %s \n" %imeiInfo['Mobile Equipment Type']
			info += u"销售市场： %s \n" %imeiInfo['Primary Market']
			droid.dialogCreateAlert(u"你的手机信息", info)
			droid.dialogShow()
		else:
			droid.makeToast(u"检索失败.可能是网络问题或不正确的IMEI号码,请重试")
	except:
		print 'Usage: python imei.py 357194040705758'
{% endhighlight %}
