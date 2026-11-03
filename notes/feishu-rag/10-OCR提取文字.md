![图片下载、私有存储、上下文提取与内容回写流程](assets/document-image-processing.png)

# OCR提取文字

OCR （Optical Character Recognition，光学字符识别）是指电子设备（例如扫描仪或数码相机）检查纸上打印的字符，通过检测暗、亮的模式确定其形状，然后用字符识别方法将形状翻译成计算机文字的过程

```python
!uv add paddlepaddle paddleocr
```

```python
image_path = "/Volumes/数据盘/课程资料/AI/deal-pilot-rag/课件/assets/document-image-processing.png"
```

```python
from paddleocr import PaddleOCR

ocr = PaddleOCR(
    use_gpu = False,
    lang="ch",
    use_angle_cls=True
)
```

```python
results = ocr.ocr(image_path)
```

```python
from rich import print
print(results)
```
