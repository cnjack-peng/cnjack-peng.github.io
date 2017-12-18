---
layout: post
title: POI两种读取Excel的方式
date: 2017-12-18 23:34:25
tags:
categories: 软件开发
---


## 前言

在实际项目上遇到了上传 Excel 并解析到数据库中的需求，经分析后选择 Apache POI 来实现，故事也就开始了……

## 常规模式

```java
File file = new File(yourFilepath);
FileInputStream is = new FileInputStream(file);
Workbook workbook = WorkbookFactory.create(is);
FormulaEvaluator formulaEvaluator = workbook.getCreationHelper().createFormulaEvaluator();

for (Sheet sheet : workbook) {
    for (Row row : sheet) {
        for (Cell cell : row) {

            // Alternatively, get the value and format it yourself
            switch (formulaEvaluator.evaluateInCell(cell).getCellTypeEnum()) {
                case STRING:
                    System.out.println(cell.getRichStringCellValue().getString());
                    break;
                case NUMERIC:
                    if (DateUtil.isCellDateFormatted(cell)) {
                        System.out.println(cell.getDateCellValue());
                    } else {
                        System.out.println(cell.getNumericCellValue());
                    }
                    break;
                case BOOLEAN:
                    System.out.println(cell.getBooleanCellValue());
                    break;
                case FORMULA:
                    System.out.println(cell.getCellFormula());
                    break;
                case BLANK:
                    System.out.println();
                    break;
                default:
                    System.out.println();
            }

        }

    }

}
```

优点：取值方便，可供调用方法多；

缺点：一次性读完值，在读取大量数据时可能会出现堆溢出。



## 事件驱动模式

优点：逐个单元格读取，理论上多大数据都能处理；

缺点：调用方法较少，需要自己动手堆数据进行处理。

查看 Excel XML 映射关系——新增 Excel 后缀名 .zip，打开 xl > worksheets > sheet1.xml ，查看对应的标签节点信息，通过 POI 提供的方法来读取。

Excel XML 片段截取：

```xml
<row r="1" spans="1:8" s="4" customFormat="1" ht="18">
    <c r="A1" s="3" t="s">
        <v>6</v>
    </c>
    <c r="B1" s="3" t="s">
        <v>7</v>
    </c>
    <c r="C1" s="3" t="s">
        <v>8</v>
    </c>
    <c r="D1" s="3" t="s">
        <v>9</v>
    </c>
    <c r="E1" s="3" t="s">
        <v>10</v>
    </c>
    <c r="F1" s="3" t="s">
        <v>11</v>
    </c>
    <c r="G1" s="3" t="s">
        <v>12</v>
    </c>
    <c r="H1" s="3" t="s">
        <v>13</v>
    </c>
</row>
```

读取 row c r t 等。

```java
public class TestEventModel {

    // EXCEL 读取文件

    public void processOneSheet(String filename) throws Exception {

        OPCPackage pkg = OPCPackage.open(filename);
        XSSFReader r = new XSSFReader(pkg);
        SharedStringsTable sst = r.getSharedStringsTable();

        XMLReader parser = fetchSheetParser(sst);
        InputStream sheet = r.getSheet("rId1");
        InputSource sheetSource = new InputSource(sheet);
        parser.parse(sheetSource);
        sheet.close();

    }

    public XMLReader fetchSheetParser(SharedStringsTable sst) throws SAXException {

        XMLReader parser = XMLReaderFactory.createXMLReader("org.apache.xerces.parsers.SAXParser");
        ContentHandler handler = new SheetHandler(sst);
        parser.setContentHandler(handler);
        return parser;
    }

    private static class SheetHandler extends DefaultHandler {

        private SharedStringsTable sst;
        private String lastContents;
        private boolean nextIsString;

        private int curRowNum = 0;
        private char nextCell = 'A';

        private Map<String, String> rowData = new HashMap<String, String>();

        private SheetHandler(SharedStringsTable sst) {
            this.sst = sst;
        }



        // 开始读取 excel xml文件中的标签
        public void startElement(String uri, String localName, String name, Attributes attributes) throws SAXException {

            // 读到了一行 <row></row>
            if (name.equals("row")) {

                // 取得当前的行数
                curRowNum = Integer.parseInt(attributes.getValue("r"));
                // 将下一列数置为第一列，也就是 excel 中的 A
                nextCell = 'A';
            }



            // 开始读列数
            if (name.equals("c")) {

                // 取得要读取的列数，值为 EXCEL 的单元格位置 A1 B1 C1 D1...
                String readCell = attributes.getValue("r");
                // 取得当前行的第一列 A1
                String curCell = nextCell + String.valueOf(curRowNum);

                // 循环判断，解决单元格空值被跳过的问题（A4为空的话，A4就不会被  attributes.getValue("r") 取到）
                // 当被跳过，单元格会错位，这里就进行判断，不相等，那么单元格列数向后 + 1 为实际单元格

                while (!curCell.equals(readCell)) {
                    // rowData——把一行单元格里的数据整合到一个 Map 中，key 为列数，值为对应的值
                    rowData.put(curCell, "");

                    // 单元格列数加一
                    nextCell = (char) ((int) nextCell + 1);

                    // 取得当前正确的单元格
                    curCell = nextCell + String.valueOf(curRowNum);
                }

                // 由于当前单元格加一，下一个单元格也要做出对应的改变
                nextCell = (char) ((int) nextCell + 1);

                // 处理单元格数据类型，全当做 string 读出来，之后对应处理
                String cellType = attributes.getValue("t");

                if (cellType != null && cellType.equals("s")) {
                    nextIsString = true;
                } else {
                    nextIsString = false;

                }
            }

            lastContents = "";

        }

        public void endElement(String uri, String localName, String name) throws SAXException {

            // 结束读取
            if (nextIsString) {

                // 取得单元格里的值作为字符串
                int idx = Integer.parseInt(lastContents);
                lastContents = new XSSFRichTextString(sst.getEntryAt(idx)).toString();
                nextIsString = false;
            }

            if (name.equals("v")) {

                // 取当前单元格的位置，并作为 key set 到 rowData 中，完成数据封装
                String curCell = (char) ((int) nextCell - 1) + String.valueOf(curRowNum);
                rowData.put(curCell, lastContents);
            }

            // 如果读到行结束
            if (name.equals("row")) {

                // 第一行为表头，不处理
                if (curRowNum == 1) {
                    return;
                }

                // 循环取 A-H 位置的值，这里要取到的值要写死在长度那，超出的不会写入到数据库
                for (int i = (int) 'A'; i <= (int) 'H'; i++) {

                    // 取当前列
                    String curCell = (char) i + String.valueOf(curRowNum);

                    // 从 rowData 里取出值
                    String data = rowData.get(curCell);

                    // 如果是单元格内是空格，会出现被置为 null 的情况，这里统一置成空字符串
                    data = data == null ? "" : data.trim();

                    switch ((char) i) {
                        case 'A':
                            // 如果 A 列是日期，能处理 yyyy/MM/DD 和 yyyy-MM-dd 两种类型
                            if (data.equals("")) {
                                // do something setDate(null);
                            } else {
                                SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
                                Date date;

                                try {
                                    date = sdf.parse(data);
                                } catch (ParseException e) {
                                    try {
                                        date = calculateDate(data);
                                    } catch (Exception e1) {
                                        // do something for error message
                                        continue;
                                    }
                                }
                                // do something setDate(date);
                            }

                            break;
                        case 'B':
                            System.out.printf(d);
                            break;
                        case 'C':
                            System.out.printf(d);
                            break;
                        case 'D':
                            System.out.printf(d);
                            break;
                        case 'E':
                            System.out.printf(d);
                            break;
                        case 'F':
                            System.out.printf(d);
                            break;
                        case 'G':
                            System.out.printf(d);
                            break;
                        case 'H':
                            System.out.printf(d);
                            break;

                    }

                }

                rowData.clear();

            }

            // 读完 excel 标志
            if (name.equals("worksheet")) {
                System.out.println("读取结束");

            }

        }

        public void characters(char[] ch, int start, int length) throws SAXException {
            lastContents += new String(ch, start, length);

        }

        private Date calculateDate(String data) throws Exception {
            if (null == data) {
                return null;
            }

            // 从 1900年到1970年的天数是25569天，减去之后再进行操作
            long millisecond = (Long.parseLong(data) - 25569) * 24 * 60 * 60 * 1000;
            if (millisecond < 0) {
                return new Date(0);
            }
            return new Date(millisecond);
        }

    }



    public static void main(String[] args) throws Exception {
        TestEventModel testEventModel = new TestEventModel();
        testEventModel.processOneSheet(yourFilePath);
    }

}
```