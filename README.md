# 利用python爬取网页内容并自动生成PDF

## 前言

python中将html转化为pdf的常用工具是Wkhtmltopdf工具包，在python环境下，pdfkit是这个工具包的封装类。如何使用pdfkit以及如何配置呢？分如下几个步骤。

1、下载wkhtmltopdf安装包，并且安装到电脑上
下载地址：<https://wkhtmltopdf.org/downloads.html>
请根据自己的系统版本，选择合适的安装包。如果没有装C语言库，建议选择Windows下的第二种。
![20180923165416.png](https://upload-images.jianshu.io/upload_images/5498924-e1770fe67d603532.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2、在pycharm中安装pdfkit库，过程就不介绍啦，前面讲过类似的内容。

3、在pycharm中安装whtmltopdf库。
这个和第一步中的安装包是两个东西，请区别开来。

## 编译环境

* Win10操作系统
* Python3.6语言版本
* Google Chrome浏览器
* Pycharm编译软件

## 目标URL

```
https://bootstrap-datepicker.readthedocs.io/en/stable/
```

## 调用库

```python
import pdfkit
import requests
from bs4 import BeautifulSoup
import os
from PyPDF2 import PdfFileReader, PdfFileWriter
```

## 目标网页解析

```python
def dealwith_html(base_url):
    try:
        r = requests.get(base_url)
        r.raise_for_status()
        r.encoding = r.apparent_encoding
        return r.text
    except:
        return ''
```

## PDF标题，链接，内容获取

```python
def get_info(base_url,html):
    chapter_info = []
    soup = BeautifulSoup(html, 'html.parser')
    global book_name
    book_name = soup.find('a', class_="icon icon-home").text.strip()
    print(book_name)
    chapters = soup.find_all('li',class_='toctree-l1')

    for chapter in chapters:
        info = {}
        info['title'] = chapter.a.text.replace('/', '').replace('*', '')
        info['url'] = base_url + chapter.a.get('href')
        info['child_chapters'] = []
        if chapter.ul is not None:
            child_chapters = chapter.ul.find_all('li')
            for child in child_chapters:
                url = child.a.get('href')

                if '#' not in url:
                    info['child_chapters'].append({
                        'title': child.a.text.replace('/', '').replace('*', ''),
                        'url': base_url + child.a.get('href'),
                    })

        chapter_info.append(info)
    return chapter_info
```

## PDF内容清洗

```python
def get_content(url):
    html_template = """
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
    </head>
    <body>
    {content}
    </body>
    </html>
    """
    html = dealwith_html(url)
    soup = BeautifulSoup(html, 'html.parser')
    content = soup.find('div', attrs='document')
    html = html_template.format(content=content)
    return html
```

## PDF文件格式保存

```python
def save_pdf(body,filename):

    options = {
        'page-size': 'Letter',
        'margin-top': '0.75in',
        'margin-right': '0.75in',
        'margin-bottom': '0.75in',
        'margin-left': '0.75in',
        'encoding': "UTF-8",
        'custom-header': [
            ('Accept-Encoding', 'gzip')
        ],
        'cookie': [
            ('cookie-name1', 'cookie-value1'),
            ('cookie-name2', 'cookie-value2'),
        ],
        'outline-depth': 10,
    }

    #注意！这里直接配置wk的地址已省去不必要麻烦
    config=pdfkit.configuration(wkhtmltopdf=r'C:\Users\dell\Desktop\wkhtmltox-0.12.5-1.mxe-cross-win64\wkhtmltox\bin\wkhtmltopdf.exe')
    pdfkit.from_string(body,filename,options=options,configuration=config)
    print('打印成功！')
```

```python
def save_to_pdf(body):
    try:
        for chapter in body:
            ctitle = chapter['title']
            url = chapter['url']
            # 文件夹不存在则创建（多级目录）
            dir_name = os.path.join(os.path.dirname(__file__), 'PDF', ctitle)
            if not os.path.exists(dir_name):
                os.makedirs(dir_name)

            html = get_content(url)

            print('保存章节：', ctitle)
            save_pdf(html, os.path.join(dir_name, ctitle + '.pdf'))

            children = chapter['child_chapters']
            if children:
                for child in children:
                    html = get_content(child['url'])
                    pdf_path = os.path.join(dir_name, child['title'] + '.pdf')
                    save_pdf(html, pdf_path)
        print('保存完毕')
    except :
        print('出现问题')
```

## 合并多个PDF为一个PDF

```python
def merge_pdf(infnList, outfn):
    pagenum = 0
    pdf_output = PdfFileWriter()

    for pdf in infnList:
        first_level_title = pdf['title']
        dir_name = os.path.join(os.path.dirname(
            __file__), 'PDF', first_level_title)
        padf_path = os.path.join(dir_name, first_level_title + '.pdf')

        pdf_input = PdfFileReader(open(padf_path, 'rb'))
        page_count = pdf_input.getNumPages()
        for i in range(page_count):
            pdf_output.addPage(pdf_input.getPage(i))
        parent_bookmark = pdf_output.addBookmark(
            first_level_title, pagenum=pagenum)
        pagenum += page_count

        if pdf['child_chapters']:
            for child in pdf['child_chapters']:
                second_level_title = child['title']
                padf_path = os.path.join(dir_name, second_level_title + '.pdf')
                pdf_input = PdfFileReader(open(padf_path, 'rb'))
                page_count = pdf_input.getNumPages()
                for i in range(page_count):
                    pdf_output.addPage(pdf_input.getPage(i))
                pdf_output.addBookmark(
                    second_level_title, pagenum=pagenum, parent=parent_bookmark)
                pagenum += page_count
    pdf_output.write(open(outfn, 'wb+'))
```

## 主函数编写

```python
if __name__ == '__main__':
    base_url = 'https://bootstrap-datepicker.readthedocs.io/en/stable/'
    html = dealwith_html(base_url)
    body = get_info(base_url,html)
    save_to_pdf(body)
    merge_pdf(body, os.path.join(os.path.dirname(__file__), book_name + '.pdf'))
```

## 结果展示

![QQ截图20180923170512.png](https://upload-images.jianshu.io/upload_images/5498924-bef6a651aa2ad735.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![QQ截图20180923170823.png](https://upload-images.jianshu.io/upload_images/5498924-61f691c4bf388721.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

已下载未合并PDF文件


![QQ截图20180923170651.png](https://upload-images.jianshu.io/upload_images/5498924-7d06d57c98b6814e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

合并后PDF文件
