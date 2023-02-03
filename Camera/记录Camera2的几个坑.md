# 记录Camera2的几个坑

#### 1. 预览拉伸
- 原因：相机输出尺寸和屏幕预览窗口宽高比不一致，预览图像被拉伸
- 解决：自定义SurfaceView，根据相机尺寸重新计算宽高，调整预览窗口大小``` kotlin
class AutoFitSurfaceView @JvmOverloads constructor(
        context: Context,
        attrs: AttributeSet? = null,
        defStyle: Int = 0
) : SurfaceView(context, attrs, defStyle) {

    private var aspectRatio = 0f

    /**
     * Sets the aspect ratio for this view. The size of the view will be
     * measured based on the ratio calculated from the parameters.
     *
     */
    fun setAspectRatio(previewSize: Size) {
        require(previewSize.width > 0 && previewSize.height > 0) { "Size cannot be negative" }
        aspectRatio = previewSize.width.toFloat() / previewSize.height.toFloat()
        holder.setFixedSize(previewSize.width, previewSize.height)

        if (aspectRatio != 0f) {
            // Performs center-crop transformation of the camera frames
            val newWidth: Int
            val newHeight: Int
            val actualRatio = if (width > height) aspectRatio else 1f / aspectRatio
            if (width < height * actualRatio) {
                newHeight = height
                newWidth = (height * actualRatio).roundToInt()
            } else {
                newWidth = width
                newHeight = (width / actualRatio).roundToInt()
            }

            Log.d(TAG, "dimensions set: $newWidth x $newHeight")
            setSize(newWidth, newHeight)
        }
//        requestLayout()
    }

/*    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec)
        val width = MeasureSpec.getSize(widthMeasureSpec)
        val height = MeasureSpec.getSize(heightMeasureSpec)
        if (aspectRatio == 0f) {
            setMeasuredDimension(width, height)
        } else {

            // Performs center-crop transformation of the camera frames
            val newWidth: Int
            val newHeight: Int
            val actualRatio = if (width > height) aspectRatio else 1f / aspectRatio
            if (width > height * actualRatio) {
                newHeight = height
                newWidth = (height * actualRatio).roundToInt()
            } else {
                newWidth = width
                newHeight = (width / actualRatio).roundToInt()
            }

            Log.d(TAG, "Measured dimensions set: $newWidth x $newHeight")
            setMeasuredDimension(newWidth, newHeight)
        }
    } */

    companion object {
        private val TAG = AutoFitSurfaceView::class.java.simpleName
    }
}
```

</br>
#### 2. 拍照镜像问题
- 前置摄像头在预览时，底层默认做了镜像翻转，保证预览时看到的画面是照镜子一样
- 拍照获取图片时，底层没有做镜像，导致看到的画面是反的
- 解决：利用Matrix.setScale(-1f, 1f)做左右镜像翻转``` kotlin 
// 设置监听imageReaderScreenshot.setOnImageAvailableListener({
            val image = it.acquireLatestImage()
            try {
                val tempBitmap =
                    ImageConvertUtils.getInstance().convertToUpRightBitmap(
                        InputImage.fromMediaImage(
                            image,
                            getRotationCompensation()
                        )
                    )
                // 镜像翻转切图
                screenshotMatrix.setScale(-1f, 1f)
                curBitmap = Bitmap.createBitmap(
                    tempBitmap,
                    0,
                    0,
                    tempBitmap.width,
                    tempBitmap.height,
                    screenshotMatrix,
                    true
                )

                tempBitmap.recycle()

            } catch (e: Exception) {
                e.printStackTrace()
            } finally {
                image?.close()
            }
        }, cameraHandler)
        
    /**
     * 拍照
     */
    private fun takePic() {
        if (cameraDevice == null || isDisableScreenshot) return

        cameraDevice?.apply {

            val captureRequestBuilder =
                createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE)
            captureRequestBuilder.addTarget(surfaceView.holder.surface)
            captureRequestBuilder.addTarget(imageReaderScreenshot.surface)

            captureRequestBuilder.set(
                CaptureRequest.CONTROL_AF_MODE,
                CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE
            ) // 自动对焦
            captureRequestBuilder.set(
                CaptureRequest.JPEG_ORIENTATION,
                getRotationCompensation()
            )      //根据摄像头方向对保存的照片进行旋转，使其为"自然方向"
            captureSession?.capture(captureRequestBuilder.build(), null, cameraHandler)
        }
    }
```

</br>
#### 3. java.lang.IllegalStateException: maxImages (2) has already been acquired, call #close before acquiring more.- Camera2拍照不能超过2张，在onnImageAvailableListener监听中拿到Image后需要将Image关掉

- onImageAvailableListener. onImageAvailable()用于接收相机图像，相机从ImageReader的队列中获取时，如果获取的Image数大于2，而中间的Image没有close，就会耗尽底层队列资源，抛出IllegalStateException- 1、拍照截图时是在子线程中完成的，由于获取到的Image需要在后面子线程里用到，这里不能直接close
- 2、由于需求调整，需要保存两张截图，最开始直接调用了两次拍照的方法，导致camera拍照超过2张没有close，后面改为拍照后保存两张截图

- Image close之后再拍下一张

</br>
#### 4. Oppo手机出现的打不开摄像头问题
- ERROR_CAMERA_DEVICE：Camera 1 error: (4) Fatal (device)  【设备遇到致命错误】

- 原因：相机的previewSize设置过大，在某些机型上不适用。- 解决：遍历相机适用的输出尺寸列表，获取小一点的而且比例合适的尺寸    - Camera自带方法getPreviewOutputSize() 获取的是最大的相机支持的尺寸，通常是屏幕宽高    - 要解决这个问题，previewSize需要比最大输出尺寸getPreviewOutputSize()小    - 为了不造成预览界面的拉伸，宽高比需要和最大输出尺寸getPreviewOutputSize()的宽高比尽量一致 

```kotlin
// 遍历找到最合适的previewSize
val map = characteristics.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP)

map?.let {
    val sizesArray = map.getOutputSizes(SurfaceTexture::class.java)
    var smallest = Size(0, 0)
    var smallDelta = Float.MAX_VALUE
    for (item in sizesArray) {
        val itemRatio = item.width.toFloat() / item.height
        val delta = abs(ratio - itemRatio)
        if (item.height < previewOutput.height && item.width < previewOutput.width && delta < 0.2 && delta < smallDelta) {
            smallest = item
            smallDelta = delta
            if (delta == 0f) {
                break
            }
        }
    }
    previewSize = smallest
}
if (previewSize.width == 0 && previewSize.height == 0) {
    previewSize = previewOutput
}
surfaceView.setAspectRatio(previewSize)
```
