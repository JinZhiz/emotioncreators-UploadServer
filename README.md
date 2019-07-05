# emotioncreators-upload-server
I社 情感工坊上传下载服务器

<br />

<h1>注意</h1>
游戏更新后，存档数据使用了 zip 进行压缩，当前代码已经不适用于新版本<br />
最新版本游戏在上传时将会压缩后再上传，下载时服务器那边也会将压缩后的数据传输给游戏，客户端会自行解压出PNG格式<br />
因为这个项目目前已经有人维护，所以这边代码就不更新了。<br />
数据压缩的方式并不复杂，只是用 zip 打包了下。<br /><br />

以下是游戏中的压缩和解压代码，使用到的是 C# 的 Ionic.Zip 代码库。<br />
具体可使用 dnSpy 反编译游戏的 Assembly-CSharp.dll 查看类 ZipAssist<br />

<br />
<pre>
<code>
public byte[] SaveUnzipFile(byte[] srcBytes, EventHandler<SaveProgressEventArgs> callBack = null)
{
	byte[] result = null;
	try
	{
		using (MemoryStream memoryStream = new MemoryStream(srcBytes))
		{
			ReadOptions options = new ReadOptions
			{
				Encoding = Encoding.GetEncoding("shift_jis")
			};
			using (ZipFile zipFile = ZipFile.Read(memoryStream, options))
			{
				ZipEntry zipEntry = zipFile[0];
				using (MemoryStream memoryStream2 = new MemoryStream())
				{
					zipEntry.Extract(memoryStream2);
					result = memoryStream2.ToArray();
				}
			}
		}
	}
	catch (Exception ex)
	{
	}
	return result;
}
</code>
</pre>

<pre>
<code>
public byte[] SaveZipBytes(byte[] srcBytes, string entryName, EventHandler<SaveProgressEventArgs> callBack = null)
{
	byte[] result = null;
	try
	{
		using (ZipFile zipFile = new ZipFile(Encoding.GetEncoding("shift_jis")))
		{
			if (callBack != null)
			{
				zipFile.SaveProgress += callBack;
			}
			else
			{
				zipFile.SaveProgress += this.SaveProgress;
			}
			zipFile.AlternateEncodingUsage = ZipOption.Always;
			zipFile.CompressionLevel = CompressionLevel.BestCompression;
			zipFile.AddEntry(entryName, srcBytes);
			long num = (long)srcBytes.Length;
			using (MemoryStream memoryStream = new MemoryStream())
			{
				if (num % 65536L == 0L)
				{
					zipFile.ParallelDeflateThreshold = -1L;
				}
				zipFile.Save(memoryStream);
				result = memoryStream.ToArray();
			}
		}
	}
	catch (Exception ex)
	{
	}
	return result;
}
</code>
</pre>

<br />

<h1>说明</h1>
因为原本游戏的服务器封国内IP，我尝试着做了这个。<br />
这些代码的功能是给 Illusion 社的 emotioncreators 当一个替代用的上传下载服务器。<br />
这些代码并没有经过测试，所以可能存在BUG<br />
我本人对PHP也是新手，做这个东西也是一边谷歌一边写的，所以不一定能提供代码上的帮助。
<br />
<br />
<br />
<h1>使用注意</h1>
如果你要使用，请注意设置 php.ini 中的 upload_max_filesize、post_max_size、memory_limit。<br />
设置到合理的允许上传文件的大小。<br />
<br />
以及 max_execution_time、max_input_time。<br />
设置一下接受上传、下载的最大时间，以免上传或下载失败。<br /><br />
正式架设时，需要把 data/config.ini 的 close_error_report 设置成 1, 避免返回奇怪的东西。
<br />
<br/>
mysql 的 sql 文件在 sql 文件夹里面。<br />
data/config.ini 可以修改 mysql 的连接配置。<br /><br />

config.ini 中 thumbnail 开头的是缩略图大小。<br />
image_base64 是将图片以 base64 编码储存。<br />
close_error_report 是关闭 php 自带报错，如果开启，可能导致游戏实际使用时出错，建议只在调试时使用。<br />
version 是游戏的版本。<br />
<br />
<br />
<br />

<h1>关于修改游戏连接</h1>
游戏连接地址，在 emotioncreators/DefaultData/url 里面<br />
里面的 .dat 就是连接地址，不过都经过加密。<br /><br />

以下是加密代码。<br /><br />
加解密工具，已经在论坛的帖子里放出，以下是加解密的 C# 代码<br />

加密 EncryptAES(bytes, "eromake", "phpaddress")<br />
解密 DecryptAES(bytes, "eromake", "phpaddress");<br />

<pre>
<code>
public static byte[] EncryptAES(byte[] srcData, string pw = "illusion", string salt = "unityunity")
{
	RijndaelManaged rijndaelManaged = new RijndaelManaged();
	rijndaelManaged.KeySize = 128;
	rijndaelManaged.BlockSize = 128;
	byte[] bytes = Encoding.UTF8.GetBytes(salt);
	Rfc2898DeriveBytes rfc2898DeriveBytes = new Rfc2898DeriveBytes(pw, bytes);
	rfc2898DeriveBytes.IterationCount = 1000;
	rijndaelManaged.Key = rfc2898DeriveBytes.GetBytes(rijndaelManaged.KeySize / 8);
	rijndaelManaged.IV = rfc2898DeriveBytes.GetBytes(rijndaelManaged.BlockSize / 8);
	ICryptoTransform cryptoTransform = rijndaelManaged.CreateEncryptor();
	byte[] result = cryptoTransform.TransformFinalBlock(srcData, 0, srcData.Length);
	cryptoTransform.Dispose();
	return result;
}

public static byte[] DecryptAES(byte[] srcData, string pw = "illusion", string salt = "unityunity")
{
	RijndaelManaged rijndaelManaged = new RijndaelManaged();
	rijndaelManaged.KeySize = 128;
	rijndaelManaged.BlockSize = 128;
	byte[] bytes = Encoding.UTF8.GetBytes(salt);
	Rfc2898DeriveBytes rfc2898DeriveBytes = new Rfc2898DeriveBytes(pw, bytes);
	rfc2898DeriveBytes.IterationCount = 1000;
	rijndaelManaged.Key = rfc2898DeriveBytes.GetBytes(rijndaelManaged.KeySize / 8);
	rijndaelManaged.IV = rfc2898DeriveBytes.GetBytes(rijndaelManaged.BlockSize / 8);
	ICryptoTransform cryptoTransform = rijndaelManaged.CreateDecryptor();
	byte[] result = cryptoTransform.TransformFinalBlock(srcData, 0, srcData.Length);
	cryptoTransform.Dispose();
	return result;
}
</code>
</pre>
