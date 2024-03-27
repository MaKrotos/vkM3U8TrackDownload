# vkM3U8TrackDownload

Этот код позволяет скачать Track из вк, если подсунуть ему ссылку на m3u8.

Для работы ему необходимо NAudio!

Пример использования:
```C#
            var downloader = new M3U8Downloader(audio.Url.ToString(), filename);
            string pathTrack = downloader.DownloadAndCombineAudio();
```

Файл получается по итогу без свойств и нормального назания.
Но достаточно сделать вот так (здесь используется либа TagLib-sharp):
```C#
  public async Task<string> DownloadTrackAsync(Audio audio, string filename) 
{
    var downloader = new M3U8Downloader(audio.Url.ToString(), filename);
    string trackPath = await downloader.DownloadAndCombineAudioAsync();

    string newPath = AddMetadataToAudioFile(trackPath, audio);

    return newPath;
}

public static string AddMetadataToAudioFile(string filePath, Audio audio)
{
    if (!System.IO.File.Exists(filePath))
    {
        Console.WriteLine($"File not found at path {filePath}");
        return filePath;
    }

    var audioFile = TagLib.File.Create(filePath);

    audioFile.Tag.Title = audio.Title;
    audioFile.Tag.Performers = new[] { audio.Artist };
    audioFile.Tag.Album = audio.Album?.Title;
    audioFile.Tag.Track = (uint)audio.Id;
    audioFile.Tag.Year = (uint)audio.Date.Year;
    audioFile.Tag.Comment = audio.Subtitle;

    audioFile.Save();

    string newFileName = $"{audio.Title} - {audio.Artist}.mp3";
    string folderPath = Path.GetDirectoryName(filePath);
    string newFilePath = System.IO.Path.Combine(folderPath, newFileName);

    System.IO.File.Move(filePath, newFilePath);
    return newFilePath;
}



```


А это сам нужный класс
```C#

 using System;
 using System.IO;
 using System.Linq;
 using System.Net;
 using System.Text;
 using System.Text.RegularExpressions;
 using NAudio.Wave;
 using System.Security.Cryptography;
 using VkNet.Model;

 public class M3U8Downloader
 {
     private string _m3u8Url;
     private string track_name;


     public M3U8Downloader(string m3u8Url, string track_name)
     {
         _m3u8Url = m3u8Url;
         this.track_name = track_name;
     }

  

     public string DownloadAndCombineAudio()
     {
         var m3u8FileContent = DownloadM3U8File();
         var audioUrls = ParseM3U8File(m3u8FileContent);
         return CombineAudioFiles(audioUrls);
     }

     private string DownloadM3U8File()
     {
         using (var client = new WebClient())
         {
             return client.DownloadString(_m3u8Url);
         }
     }


     public class AudioSegment
     {
         public string Url { get; set; }
         public string KeyUrl { get; set; }

         public int sq { get; set; }
     }

     int itemn = 1;

     private List<AudioSegment> ParseM3U8File(string m3u8FileContent)
     {
         var lines = m3u8FileContent.Split('\n');
         var audioSegments = new List<AudioSegment>();
         string keyUrl = null;

         foreach (var line in lines)
         {
             if (line.StartsWith("#EXT-X-KEY:METHOD=AES-128"))
             {
                 var match = Regex.Match(line, "URI=\"(.*?)\"");
                 if (match.Success)
                 {
                     keyUrl = match.Groups[1].Value;
                 }
             }
             else if (line.StartsWith("#EXT-X-KEY:METHOD=NONE"))
             {
                 keyUrl = null;
             }
             else if (!line.StartsWith("#") && !string.IsNullOrWhiteSpace(line))
             {
                 audioSegments.Add(new AudioSegment { Url = line.Trim(), KeyUrl = keyUrl, sq = itemn++ });

             }
         }
         return audioSegments;
     }





     public string DownloadFileAndReadText(string url)
     {
         using (WebClient client = new WebClient())
         {
             using (Stream stream = client.OpenRead(url))
             {
                 using (StreamReader reader = new StreamReader(stream))
                 {
                     string text = reader.ReadToEnd();
                     return text;
                 }
             }
         }
     }


     private string CombineAudioFiles(List<AudioSegment> audioSegments)
     {
         string path = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "segments", track_name);
         string pat = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "segments");
         if (!Directory.Exists(path)) Directory.CreateDirectory(path);
         byte[] buffer = new byte[1024];

         using (WaveFileWriter waveFileWriter = new WaveFileWriter(pat + "\\"+track_name+".mp3", new WaveFormat()))
         {

             foreach (var segment in audioSegments)
             {
                 var audioData = DownloadAudio(_m3u8Url.Replace("index.m3u8", "") + segment.Url);
                 if (segment.KeyUrl != null)
                 {

                     var lastKey = DownloadFileAndReadText(segment.KeyUrl);
                     audioData = DecryptFile(audioData, lastKey, segment.sq);
                     if (audioData == null) continue;
                 }

                 
                  using (FileStream fs = new FileStream(path + "\\" + segment.Url + ".mp3", FileMode.Create, FileAccess.Write))
                 {
                     fs.Write(audioData, 0, audioData.Length);
                 }
                 using (MediaFoundationReader reader = new MediaFoundationReader(path + "\\" + segment.Url + ".mp3"))
                 {
                     int bytesRead;
                     while ((bytesRead = reader.Read(buffer, 0, buffer.Length)) > 0)
                     {
                         waveFileWriter.Write(buffer, 0, bytesRead);
                     }
                  
                 }
                 File.Delete(path + "\\" + segment.Url + ".mp3");
             }
             Directory.Delete(path);
         }

         return pat + "\\" + track_name + ".mp3";
     }


     private byte[] DownloadAudio(string url)
     {
         using (var client = new WebClient())
         {
             return client.DownloadData(url);
         }
     }


     public static byte[] tobigendianbytes(int i)
     {
         byte[] bytes = BitConverter.GetBytes(Convert.ToInt64(i));
         if (BitConverter.IsLittleEndian)
             Array.Reverse(bytes);
         return bytes;
     }


     public static byte[] DecryptFile(byte[] cipherTextBytes, string key, int ivv)
     {
         byte[] plainTextBytes = null;

         using (Aes aes = Aes.Create())
         {
           
             byte[] iv = tobigendianbytes((int)ivv);
             iv = new byte[8].Concat(iv).ToArray();

             var keyBytes = new byte[16];
             var secretKeyBytes = Encoding.UTF8.GetBytes(key);
             Array.Copy(secretKeyBytes, keyBytes, Math.Min(keyBytes.Length, secretKeyBytes.Length));
             aes.Mode = CipherMode.CBC;
             aes.Key = keyBytes;

             aes.IV = iv; // AES needs a 16-byte IV

             // Create a decryptor
             ICryptoTransform decryptor = aes.CreateDecryptor(aes.Key, aes.IV);

             using (MemoryStream ms = new MemoryStream(cipherTextBytes))
             {
                 using (CryptoStream cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read))
                 {
                     plainTextBytes = new byte[cipherTextBytes.Length];

                     int decryptedByteCount = cs.Read(plainTextBytes, 0, plainTextBytes.Length);
                     ms.Close();
                     cs.Close();
                 }
             }
         }

         return plainTextBytes;
     }
    
       

    }


```
