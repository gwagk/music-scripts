# Ubuntu YouTube Music Downloader & Flash Drive Workflow

คู่มือการดาวน์โหลดเพลงจาก YouTube Playlist จัดระเบียบแยกโฟลเดอร์ศิลปิน และย้ายลงแฟลชไดรฟ์อย่างปลอดภัยบน Ubuntu

## 1. ติดตั้งเครื่องมือ
sudo apt update && sudo apt install -y yt-dlp ffmpeg

## 2. ดาวน์โหลดเพลงจาก Playlist
yt-dlp -x --audio-format mp3 --audio-quality 0 -o "~/Music/%(title)s.%(ext)s" --yes-playlist "<URL_PLAYLIST>"

## 3. สคริปต์แยกโฟลเดอร์ตามชื่อศิลปิน (Artist - Title.mp3)
cd ~/Music
for f in *" - "*; do
  if [ -f "$f" ]; then
    artist="${f%% - *}"
    mkdir -p "$artist"
    mv -v "$f" "$artist/"
  fi
done

## 4. ย้ายเข้าแฟลชไดรฟ์และบันทึกข้อมูล
sudo mkdir -p /media/gwagk/MUSIC
sudo mount -o umask=000 /dev/sda1 /media/gwagk/MUSIC
cd ~/Music
find . -type f \( -iname "*.mp3" -o -iname "*.m4a" -o -iname "*.wav" -o -iname "*.flac" \) -exec cp -v --parents "{}" /media/gwagk/MUSIC/ \;
sync
find ~/Music -type f \( -iname "*.mp3" -o -iname "*.m4a" -o -iname "*.wav" -o -iname "*.flac" \) -delete
find ~/Music -mindepth 1 -type d -empty -delete
sudo umount /media/gwagk/MUSIC
