---
layout: post
title: Python处理excel读写
date: 2016-09-30 18:00:00 +0800
tags:
- python
- xlrd
- xlwt
---

Python的`xlrd`和`xlwt`库支持了Excel文件的读写，本文列举了其简单的使用方法。

**Read Excel**

{% highlight python %}
import xlrd
import json

def open_excel(file = 'file.xlsx')
	try:
		book = xlrd.open_workbook(file)
		return book
	except Exception, e:
		print str(e)

def excel_table_by_index(file = 'file.xlsx', by_index = 0):
	data = open_excel(file)
	table = data.sheets()[by_index]
	nrows = table.nrows
	ncols = table.ncols
	lst = []
	for rownum in range(1, nrows):
		row = table.row_values(rownum)

		if row:
			app = []
			for i in range(ncols):
				app.append(row[i])
		lst.append(app)
	return json.dumps(lst, ensure_ascii = False)

def excel_table_byname(file = 'file.xls', by_name = '内容'):
	data = open_excel(file)
	# 这里需要对utf-8格式的中文代码进行转换，转换成Unicore
	table = data.sheet_by_name(unicode(by_name, 'utf-8'))
	nrows = table.nrows
	ncols = table.ncols
	lst = []
	for rownum in range(1, nrows):
		row = table.row_values(rownum)

		if row:
			app = []
			for i in range(ncols):
				app.append(row[i])
		lst.append(app)
	return json.dumps(lst, ensure_ascii = False)
{% endhighlight %}

**Write Excel**

{% highlight python %}
import csv
import xlwt

xlst = xlwt.Workbook()

def csv2xlsx(xlsx, log):
	try:
		data = os.path.basename(log).split('_')
		sheetname = "%s_%s" % (data[0], data[1])
		sheet = xlsx.add_sheet(sheetname)
	except Exception, e:
		print("save %s to xlsx error: %s" % (log, e)) 
		return
	try:
		csvfile = open(log, "rb")
		reader = csv.reader(csvfile)
		l = 0 
		for line in reader:
			if line[0].startswith('#'):
				continue
			r = 0 
			for i in line:
				sheet.write(l, r, i)
				r += 1
			l += 1
	except Exception, e:
		print("write %s to xlsx error: %s" % (log, e)) 
	csvfile.close()
		print("save %s to %s" % (log, sheetname))
	os.remove(log)
{% endhighlight %}
