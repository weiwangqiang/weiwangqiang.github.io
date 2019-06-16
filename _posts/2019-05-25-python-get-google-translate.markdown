---
layout:     post
title:      "用python爬Google翻译"
subtitle:   " \"借用Google翻译接口用用\""
date:       2019-06-16 21:10:00
author:     "Weiwq"
header-img: "img/background/post-bg-re-vs-ng2.jpg"
catalog: true
tags:
    - python
---

> “爬Google翻译还是需要点技巧的“


## 引言

在做全球应用中，正确处理每个国家的翻译是必不可少的，也是最棘手的问题，尤其是那些文字方向是右到左的。为了保证翻译是正确的，这里需要借用Google翻译的接口，为此，特意写了一个python脚本来对接google翻译 ^_^。

## 1、GoogleTranslate代码(入口代码)

```java
#!/usr/bin/python
# -*- encoding:utf-8 -*-
import requests
from urllib.parse import quote

import TkParams
import json

cookies = [
    '_ga=GA1.3.1058806942.1531796090; NID=182=k5XIOE2zGfcgE2KEP4iTQWpzGcWXQpEHZhBh_3BF9lRwlnxyn24W2jnTaXfadXinVn6ZVa4Mkpk8HZS02sF7adR-6XI60kfQMEut5c9VQgZxDfgJnatiVzhS7qrHyZ4zP3bamIWHZ16BxtOPfiLeAsgxbUu9g_0XzqSAqgQp9GI; _gid=GA1.3.1828501068.1558528281; 1P_JAR=2019-5-23-0; _gat=1',
    '_ga=GA1.3.1058806942.1531796090; _gid=GA1.3.366383484.1556091451; NID=182=k5XIOE2zGfcgE2KEP4iTQWpzGcWXQpEHZhBh_3BF9lRwlnxyn24W2jnTaXfadXinVn6ZVa4Mkpk8HZS02sF7adR-6XI60kfQMEut5c9VQgZxDfgJnatiVzhS7qrHyZ4zP3bamIWHZ16BxtOPfiLeAsgxbUu9g_0XzqSAqgQp9GI; 1P_JAR=2019-4-25-0; _gat=1'
]


class constant:
    cookie_index = 0


"""
get translate url 
"""


def _get_translate_url(from_language, to_language, translate_text):
    TkParams.refresh_tkk()
    tk = TkParams.acquire(translate_text)
    key = quote(translate_text)
    sl = from_language
    tl = to_language
    url = 'https://translate.google.cn/translate_a/single?client=webapp&sl=%s&tl=%s&hl=zh-CN&dt=at&dt=bd&dt=ex&dt=ld&dt=md&dt=qca&dt=rw&dt=rm&dt=ss&dt=t&otf=1&pc=1&ssel=0&tsel=0&kc=3&tk=%s&q=%s' % (
        sl, tl, tk, key)
    return url


def _get(url):
    cookie_len = len(cookies)
    constant.cookie_index += 1
    constant.cookie_index %= cookie_len
    headers = {'content-type': 'application/json; charset=UTF-8',
               'accept-language': 'zh-CN,zh;q=0.9',
               "content-disposition": "attachment; filename=\"f.txt\"",
               'Accept-Encoding': '',
               "cookie": cookies[constant.cookie_index],
               'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36',
               "accept": "text/html",
               "x-client-data": "CJe2yQEIo7bJAQjEtskBCKmdygEIqKPKAQixp8oBCOKoygEI8anKAQivrMoBGIKYygE=",
               "accept-Charset": "UTF-8"}
    try:
        res = requests.get(url, verify=True, headers=headers, timeout=3)
    except:
        return '', 404
    return res.text.encode('UTF-8').decode('UTF-8'), res.status_code


"""
获取结果列表的第一个翻译
"""


def _getTranslateFirstWord(translateJson):
    return translateJson[0][0][0].lower()


"""
翻译接口，提供外部使用
参数说明：

#from_language: 单词的语言代码，如：'auto'（自动检测语言）
#to_language: 目标语言代码，如：'en' (英语)
#word: 需要翻译的文字 
"""

def translate(from_language, to_language, word):
    url = _get_translate_url(from_language, to_language, word)
    translateRes, code = _get(url)
    if code is not 200:
        return ''
    jsonArray = json.loads(translateRes.lower())
    return _getTranslateFirstWord(jsonArray)


if __name__ == '__main__':
    print(translate('auto', 'en', '你好'))

```

##2、翻译鉴权
由于Google翻译的接口是收费的，会对翻译内容做一个加密鉴权的限制，可以通过一下方法得到翻译词条的加密字符串，TkParams类的实现如下：

```java
#!/usr/bin/python
# -*- encoding:utf-8 -*-
import math
import re
import requests


class constant:
    TKK = '432247.1200919766'  # this value maybe change by google ,then we need update


def rshift(val, n):
    """python port for '>>>'(right shift with padding)
    """
    return (val % 0x100000000) >> n


def _xr(a, b):
    size_b = len(b)
    c = 0
    while c < size_b - 2:
        d = b[c + 2]
        d = ord(d[0]) - 87 if 'a' <= d else int(d)
        d = rshift(a, d) if '+' == b[c + 1] else a << d
        a = a + d & 4294967295 if '+' == b[c] else a ^ d

        c += 3
    return a


def acquire(text):
    a = []
    # Convert text to ints
    for i in text:
        val = ord(i)
        if val < 0x10000:
            a += [val]
        else:
            # Python doesn't natively use Unicode surrogates, so account for those
            a += [
                math.floor((val - 0x10000) / 0x400 + 0xD800),
                math.floor((val - 0x10000) % 0x400 + 0xDC00)
            ]

    b = constant.TKK
    d = b.split('.')
    b = int(d[0]) if len(d) > 1 else 0

    # assume e means char code array
    e = []
    g = 0
    size = len(text)
    while g < size:
        l = a[g]
        # just append if l is less than 128(ascii: DEL)
        if l < 128:
            e.append(l)
        # append calculated value if l is less than 2048
        else:
            if l < 2048:
                e.append(l >> 6 | 192)
            else:
                # append calculated value if l matches special condition
                if (l & 64512) == 55296 and g + 1 < size and \
                        a[g + 1] & 64512 == 56320:
                    g += 1
                    l = 65536 + ((l & 1023) << 10) + (a[g] & 1023)  # This bracket is important
                    e.append(l >> 18 | 240)
                    e.append(l >> 12 & 63 | 128)
                else:
                    e.append(l >> 12 | 224)
                e.append(l >> 6 & 63 | 128)
            e.append(l & 63 | 128)
        g += 1
    a = b
    for i, value in enumerate(e):
        a += value
        a = _xr(a, '+-a^+6')
    a = _xr(a, '+-3^+b+-f')
    a ^= int(d[1]) if len(d) > 1 else 0
    if a < 0:  # pragma: nocover
        a = (a & 2147483647) + 2147483648
    a %= 1000000  # int(1E6)

    return '{}.{}'.format(a, a ^ b)


def _get(url):
    headers = {'content-type': 'application/json; charset=UTF-8',
               'accept-language': 'zh-CN,zh;q=0.9',
               "content-disposition": "attachment; filename=\"f.txt\"",
               'Accept-Encoding': '',
               "cookie": "_ga=GA1.3.1058806942.1531796090; _gid=GA1.3.743573119.1555898451; 1P_JAR=2019-4-22-2; NID=181=B4bCvuoYOtOPbrQB1626zFADiTQkwCg7F8AYSi1heEAi08NXZGTrYLqDyqjvmX3O_wzaVDq2SBk9gIQE2aKFiGOQqCW6PEcsHyH9gvEqnXa81gq0fhjsq_5UNXY2JWTdRdQB_Da9sAHLG-S5vcDx0SxMvx9qcX--hJmWcYzVLeU",
               'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36',
               "accept": "text/html",
               "accept-Charset": "UTF-8"}
    try:
        res = requests.get(url, verify=True, headers=headers, timeout=3)
    except:
        return '', 404
    return res.text.encode('UTF-8').decode('UTF-8'), res.status_code


# refresh tkk key
def refresh_tkk():
    text, code = _get("https://translate.google.cn/")
    if code != 200:
        return False
    text = re.findall(r'tkk(.+?),', text)
    constant.TKK = re.sub(':|\'', '', text[0])
    return True
```

##3、语言代码对照表
为了方便使用，这里整理了多国语言对应的缩写，应是比较全了, LanguageMapCode类如下

```java
languageMapCode = {
    '检测语言': 'auto',
    '阿尔巴尼亚语': 'sq',
    '阿拉伯语': 'ar',
    '阿姆哈拉语': 'am',
    '阿塞拜疆语': 'az',
    '爱尔兰语': 'ga',
    '爱沙尼亚语': 'et',
    '巴斯克语': 'eu',
    '白俄罗斯语': 'be',
    '保加利亚语': 'bg',
    '冰岛语': 'is',
    '波兰语': 'pl',
    '波斯尼亚语': 'bs',
    '波斯语': 'fa',
    '布尔语(南非荷兰语)': 'af',
    '丹麦语': 'da',
    '德语': 'de',
    '俄语': 'ru',
    '法语': 'fr',
    '菲律宾语': 'tl',
    '芬兰语': 'fi',
    '弗里西语': 'fy',
    '高棉语': 'km',
    '格鲁吉亚语': 'ka',
    '古吉拉特语': 'gu',
    '哈萨克语': 'kk',
    '海地克里奥尔语': 'ht',
    '韩语': 'ko',
    '豪萨语': 'ha',
    '荷兰语': 'nl',
    '吉尔吉斯语': 'ky',
    '加利西亚语': 'gl',
    '加泰罗尼亚语': 'ca',
    '捷克语': 'cs',
    '卡纳达语': 'kn',
    '科西嘉语': 'co',
    '克罗地亚语': 'hr',
    '库尔德语': 'ku',
    '拉丁语': 'la',
    '拉脱维亚语': 'lv',
    '老挝语': 'lo',
    '立陶宛语': 'lt',
    '卢森堡语': 'lb',
    '罗马尼亚语': 'ro',
    '马尔加什语': 'mg',
    '马耳他语': 'mt',
    '马拉地语': 'mr',
    '马拉雅拉姆语': 'ml',
    '马来语': 'ms',
    '马其顿语': 'mk',
    '毛利语': 'mi',
    '蒙古语': 'mn',
    '孟加拉语': 'bn',
    '缅甸语': 'my',
    '苗语': 'hmn',
    '南非科萨语': 'xh',
    '南非祖鲁语': 'zu',
    '尼泊尔语': 'ne',
    '挪威语': 'no',
    '旁遮普语': 'pa',
    '葡萄牙语': 'pt',
    '普什图语': 'ps',
    '齐切瓦语': 'ny',
    '日语': 'ja',
    '瑞典语': 'sv',
    '萨摩亚语': 'sm',
    '塞尔维亚语': 'sr',
    '塞索托语': 'st',
    '僧伽罗语': 'si',
    '世界语': 'eo',
    '斯洛伐克语': 'sk',
    '斯洛文尼亚语': 'sl',
    '斯瓦希里语': 'sw',
    '苏格兰盖尔语': 'gd',
    '宿务语': 'ceb',
    '索马里语': 'so',
    '塔吉克语': 'tg',
    '泰卢固语': 'te',
    '泰米尔语': 'ta',
    '泰语': 'th',
    '土耳其语': 'tr',
    '威尔士语': 'cy',
    '乌尔都语': 'ur',
    '乌克兰语': 'uk',
    '乌兹别克语': 'uz',
    '西班牙语': 'es',
    '希伯来语': 'iw',
    '希腊语': 'el',
    '夏威夷语': 'haw',
    '信德语': 'sd',
    '匈牙利语': 'hu',
    '修纳语': 'sn',
    '亚美尼亚语': 'hy',
    '伊博语': 'ig',
    '意大利语': 'it',
    '意第绪语': 'yi',
    '印地语': 'hi',
    '印尼巽他语': 'su',
    '印尼语': 'id',
    '印尼爪哇语': 'jw',
    '英语': 'en',
    '约鲁巴语': 'yo',
    '越南语': 'vi',
    '中文': 'zh-CN',
    '中文(繁体)': 'zh-TW'
}


"""
获取语言代码
"""

def get_language_code(language):
    if language in languageMapCode:
        return languageMapCode[language]
    return ''

```

用法如下：

```java
if __name__ == '__main__':
	to_language = LanguageMapCode.get_language_code('英语')
    print(GoogleTranslate.translate('auto', to_language, '你好'))
```

     
## 后记
 
处理多国翻译的工作，最好还是交给专业的翻译公司，当然，也可以参考Google翻译，毕竟Google的产品还是比较靠谱的～～


—— Weiwq 后记于 2019.06 广州


