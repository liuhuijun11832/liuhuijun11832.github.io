---
title: Java输出文字到图片
categories: 编程技术
date: 2019-01-30 16:30:59
tags:
- Java
- 杂记
---
# 描述
java 的awt工具包里是有图像类(Image)，所以可以读取图像并利用Graphics2D进行操作。但是ImageIO.read()方法读取图片时可能存在不正确处理图片ICC（ICC为JPEG图片格式中的一种头部信息）信息的问题，导致渲染图片前景色时蒙上一层红色，因此需要用JDK提供的Toolkit.getDefaultToolkit()进行操作。
<!--more-->

# 代码
```java
	import javax.imageio.ImageIO;
	import javax.swing.*;
	import java.awt.*;
	import java.awt.image.BufferedImage;
	import java.io.ByteArrayOutputStream;
	import java.io.FileOutputStream;
	import java.net.URL;
	
	public class PrintImageUtil {
	    
	    public String printWordImage(String word,String imgUrl){
	        String imgStr = "";
	        try {
	            URL url = new URL(imgUrl);
	            Image src = Toolkit.getDefaultToolkit().createImage(url);
	            BufferedImage bi = toBufferedImage(src);
	            if(bi != null){
	                g = bi.createGraphics();
	                g.setBackground(Color.WHITE);
	                g.setColor(Color.WHITE);
	                g.setFont(new Font("黑体",Font.ITALIC,50));
	                g.drawString(word,130,250);
	                g.setFont(new Font("黑体",Font.BOLD,15));
	                g.drawString("兔比",130+26*word.length(),250);
	                g.dispose();
	                ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
	                ImageIO.write(bi,"png",outputStream);
	                byte[] bytes = outputStream.toByteArray();
	                //该bytes数组就可以输出到文件，或者云服务器
	            }
	        } catch (Exception e) {
	            e.printStackTrace();
	        }
	        return "http://img.yiqx.com/"+imgStr;
	    }
	    
	    public static BufferedImage toBufferedImage(Image image) {
	        if (image instanceof BufferedImage) {
	            return (BufferedImage) image;
	        }
	        // This code ensures that all the pixels in the image are loaded
	        image = new ImageIcon(image).getImage();
	        BufferedImage bimage = null;
	        GraphicsEnvironment ge = GraphicsEnvironment
	                .getLocalGraphicsEnvironment();
	        try {
	            int transparency = Transparency.OPAQUE;
	            GraphicsDevice gs = ge.getDefaultScreenDevice();
	            GraphicsConfiguration gc = gs.getDefaultConfiguration();
	            bimage = gc.createCompatibleImage(image.getWidth(null),
	                    image.getHeight(null), transparency);
	        } catch (HeadlessException e) {
	            // The system does not have a screen
	        }
	        if (bimage == null) {
	            // Create a buffered image using the default color model
	            int type = BufferedImage.TYPE_INT_RGB;
	            bimage = new BufferedImage(image.getWidth(null),
	                    image.getHeight(null), type);
	        }
	        // Copy image to buffered image
	        Graphics g = bimage.createGraphics();
	        // Paint the image onto the buffered image
	        g.drawImage(image, 0, 0, null);
	        g.dispose();
	        return bimage;
	    }
	}```

# 一些问题
当代码打包上传到Linux服务器的时候，某些字体就无法使用了，所以需要到Winsows下找到对应的字体，最笨的办法就是复制所有的字体，上传到Linux服务器。服务器路径为/usr/java/jdk1.8.0_161/jre/lib/fonts(添加jdk支持的字体)/或者/usr/share/fonts/（添加系统字体）。