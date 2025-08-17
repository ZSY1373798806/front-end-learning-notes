### canvas图片压缩

[pFktpoKk - 码上掘金](https://code.juejin.cn/pen/7538599365147426826)

```typescript
/**
 * 图片压缩工具类
 * 提供图片压缩功能，支持调整质量、尺寸和格式转换
 * 
 * 功能特点：
 * 1. 支持多种图片格式（JPEG、PNG、WebP）
 * 2. 可调整压缩质量（0-1）
 * 3. 可设置最大尺寸（长边）
 * 4. 智能处理PNG格式（超过阈值返回原图）
 * 5. 自动生成压缩后的文件名
 * 
 * 使用示例：
 * const compressor = new Compressor({
 *   file: imageFile,
 *   quality: 0.8,
 *   mineType: 'image/jpeg',
 *   size: 1200,
 *   convertSize: 1024 * 1024 // 1MB
 * });
 * const compressedFile = await compressor.compress();
 */

// 压缩选项接口定义
interface CompressorOptions {
  file: File;          // 要压缩的图片文件对象
  quality?: number;    // 压缩质量 (0-1]，默认0.8
  mineType?: string;   // 压缩后的图片MIME类型，默认'image/jpeg'
  size?: number;       // 压缩后图片的最大尺寸（长边），默认1200px
  convertSize?: number;// PNG格式转换阈值（单位字节），默认1MB
}

class Compressor {
  // 匹配图片MIME类型的正则表达式
  static REGEXP_IMAGE_TYPE: RegExp = /^image\//;

  // 默认压缩选项
  static DEFAULT_OPTIONS: CompressorOptions = {
    file: null as any, // 初始化为null，但在使用前会被实际文件覆盖
    quality: 0.8,
    mineType: 'image/jpeg',
    size: 1200,
    convertSize: 1024 * 1024 // 1MB
  };

  // 压缩选项
  private options: CompressorOptions;
  // 要压缩的文件
  private file: File;

  /**
   * 构造函数
   * @param options 压缩选项
   * @throws 如果传入的文件不是图片类型，抛出错误
   */
  constructor(options: CompressorOptions) {
    // 合并用户选项和默认选项
    this.options = Object.assign({}, Compressor.DEFAULT_OPTIONS, options);
    this.file = this.options.file;

    // 验证文件是否为图片
    if (!this.file || !this.isImageType(this.file.type)) {
      throw new Error('请选择有效的图片文件');
    }

    // 如果用户指定的输出类型不是图片类型，则使用原文件类型
    if (!this.isImageType(this.options.mineType as string)) {
      this.options.mineType = this.file.type;
    }

    // 验证压缩质量是否在有效范围内
    if (this.options.quality && (this.options.quality > 1 || this.options.quality <= 0)) {
      this.options.quality = 0.8; // 使用默认值
    }

    // 确保尺寸是整数
    if (this.options.size) {
      this.options.size = Number.parseInt(this.options.size.toString());
    }
  }

  /**
   * 执行图片压缩
   * @returns 压缩后的File对象
   * @throws 压缩过程中发生错误时抛出异常
   */
  async compress(): Promise<File> {
    try {
      // 检查浏览器是否支持canvas.toBlob方法
      if (!HTMLCanvasElement.prototype.toBlob) {
        console.warn('当前浏览器不支持canvas.toBlob方法，返回原始文件');
        return this.file;
      }

      // 步骤1: 将File对象转换为Image对象
      const image = await this.file2Image();

      // 步骤2: 计算压缩后的尺寸
      const { width, height } = this.getExpectedEdge(image);

      // 步骤3: 将图片绘制到Canvas
      const canvas = this.image2Canvas(image, width, height);

      // 步骤4: 将Canvas转换为Blob
      const blob = await this.canvas2Blob(canvas);

      // 特殊处理：对于PNG格式且压缩后大小超过阈值的情况，返回原图
      if (this.options.mineType === 'image/png' && blob.size > (this.options.convertSize || 0)) {
        console.log('PNG图片压缩后大小超过阈值，返回原图');
        return this.file;
      }

      // 步骤5: 创建新的File对象并返回
      return new File([blob], this.getCovertName(), {
        type: this.options.mineType
      });
    } catch (error) {
      console.error('图片压缩失败:', error);
      throw error;
    }
  }

  /**
   * 将Canvas转换为Blob
   * @param canvas HTMLCanvasElement对象
   * @returns 包含图片数据的Blob对象
   * @throws 如果转换失败抛出错误
   */
  private canvas2Blob(canvas: HTMLCanvasElement): Promise<Blob> {
    return new Promise((resolve, reject) => {
      // 使用canvas的toBlob方法生成Blob
      canvas.toBlob(
        blob => {
          if (blob) {
            resolve(blob);
          } else {
            reject('Blob转换失败');
          }
        },
        this.options.mineType,  // 指定MIME类型
        this.options.quality    // 指定压缩质量
      );
    });
  }

  /**
   * 将图片绘制到Canvas
   * @param image HTMLImageElement对象
   * @param width 目标宽度
   * @param height 目标高度
   * @returns 绘制好的Canvas元素
   * @throws 如果无法获取2D上下文抛出错误
   */
  private image2Canvas(image: HTMLImageElement, width: number, height: number): HTMLCanvasElement {
    // 创建Canvas元素
    const canvas = document.createElement('canvas');
    canvas.width = width;
    canvas.height = height;

    // 获取2D渲染上下文
    const ctx = canvas.getContext('2d');
    if (!ctx) {
      throw new Error('无法获取Canvas 2D上下文');
    }

    // 对于JPEG格式，填充白色背景（解决透明区域变黑问题）
    if (this.options.mineType === 'image/jpeg') {
      ctx.fillStyle = '#FFFFFF';
      ctx.fillRect(0, 0, canvas.width, canvas.height);
    }

    // 绘制图片到Canvas
    ctx.drawImage(image, 0, 0, width, height);
    return canvas;
  }

  /**
   * 计算压缩后的图片尺寸
   * @param image HTMLImageElement对象
   * @returns 包含宽度和高度的对象
   */
  private getExpectedEdge(image: HTMLImageElement): { width: number; height: number } {
    const { naturalWidth, naturalHeight } = image;

    // 如果没有指定尺寸，则使用原始尺寸
    if (!this.options.size) {
      return { width: naturalWidth, height: naturalHeight };
    }

    // 计算宽高比
    const aspectRatio = naturalWidth / naturalHeight;
    let width: number, height: number;

    // 根据宽高比计算新尺寸
    if (aspectRatio > 1) {
      // 横向图片（宽 > 高）
      width = this.options.size;
      height = Math.round(this.options.size / aspectRatio);
    } else {
      // 纵向图片（高 >= 宽）
      width = Math.round(this.options.size * aspectRatio);
      height = this.options.size;
    }

    return { width, height };
  }

  /**
   * 将File对象转换为Image对象
   * @returns 加载完成的Image对象
   * @throws 如果图片加载失败抛出错误
   */
  private file2Image(): Promise<HTMLImageElement> {
    return new Promise((resolve, reject) => {
      // 获取URL对象（兼容不同浏览器）
      const URL = window.URL || window.webkitURL;
      const image = new Image();

      // 设置图片加载失败的回调
      image.onerror = () => {
        URL.revokeObjectURL(image.src); // 释放内存
        reject('图片加载失败');
      };

      // 设置图片加载完成的回调
      image.onload = () => {
        URL.revokeObjectURL(image.src); // 释放内存
        resolve(image);
      };

      // 创建Object URL并加载图片
      image.src = URL.createObjectURL(this.file);
    });
  }

  /**
   * 检查是否为图片类型
   * @param value MIME类型字符串
   * @returns 是否是图片类型
   */
  private isImageType(value: string): boolean {
    return Compressor.REGEXP_IMAGE_TYPE.test(value);
  }

  /**
   * 获取转换后的文件名
   * @returns 新文件名（在原文件名后添加_compressed.扩展名）
   */
  private getCovertName(): string {
    // 获取文件名（不含扩展名）
    const lastDotPos = this.file.name.lastIndexOf('.');
    const baseName = lastDotPos > 0
      ? this.file.name.substring(0, lastDotPos)
      : this.file.name;

    // 根据输出格式确定文件扩展名
    let extension = 'jpg';
    if (this.options.mineType === 'image/png') {
      extension = 'png';
    } else if (this.options.mineType === 'image/webp') {
      extension = 'webp';
    }

    // 返回新文件名
    return `${baseName}_compressed.${extension}`;
  }
}
```

方法调用

```typescript
// 处理文件选择事件
  fileInput.addEventListener('change', (e: Event) => {
    const target = e.target as HTMLInputElement;
    if (!target.files || target.files.length === 0) return;

    const file = target.files[0];

    // 验证文件是否为图片
    if (!file.type.match(Compressor.REGEXP_IMAGE_TYPE)) {
      alert('请选择有效的图片文件 (JPG, PNG, GIF, WebP)');
      return;
    }

    originalFile = file;
    showOriginalImage(file);
  });

  // 处理压缩按钮点击事件
  compressBtn.addEventListener('click', async () => {
    if (!originalFile) {
      alert('请先选择图片');
      return;
    }

    try {
      // 显示加载状态
      toggleSpinner(true);

      // 创建压缩器实例
      const compressor = new Compressor({
        file: originalFile,
        quality: parseInt(qualitySlider.value) / 100, // 转换为0-1范围
        mineType: formatSelect.value,
        size: parseInt(sizeSlider.value),
        convertSize: 1024 * 1024 // 1MB
      });

      // 执行压缩
      compressedFile = await compressor.compress();

      // 显示压缩结果
      showCompressedImage(compressedFile);

      // 显示压缩对比信息
      showComparison(originalFile, compressedFile);
    } catch (error) {
      console.error('压缩失败:', error);
      alert(`压缩失败: ${(error as Error).message || error}`);
    } finally {
      // 隐藏加载状态
      toggleSpinner(false);
    }
  });

  // 处理下载按钮点击事件
  downloadBtn.addEventListener('click', () => {
    if (!compressedFile) return;

    // 创建下载链接并触发下载
    const url = URL.createObjectURL(compressedFile);
    const a = document.createElement('a');
    a.href = url;
    a.download = compressedFile.name;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);

    // 释放URL对象
    URL.revokeObjectURL(url);
  });
```

