# MP4_report_39

## Team Member & Contributions

 * ## **107062301 李昆諭**
 * ## **107060017 廖凰汝**

| 工作項目   | 分工            |
| ---------- | --------------- |
| Trace Code | 李昆諭 |
| 報告撰寫 | both |
| 功能實作   |  廖凰汝  |

# Trace code

用state disgram幫助理解:

![](https://imgur.com/a/DEXrlgL)

# Q1
Free block儲存是用 **PersistentBitmap** 來實作的

1.  一開始在FileSystem::FileSystem先create new freemap和它的header
```javascript=
PersistentBitmap *freeMap = new PersistentBitmap(NumSectors);
FileHeader *mapHdr = new FileHeader;
```
2. 接下來將FreeMapSector標記為已使用
```javascript=
freeMap->Mark(FreeMapSector);	    
```
3. 將fileheader allocate在FreeMapSector(0)上
```javascript=
ASSERT(mapHdr->Allocate(freeMap, FreeMapFileSize));
mapHdr->WriteBack(FreeMapSector);
```
4.freemap 中由1和0組成，若為1代表已被占用若為0代表為未使用，可以透過freeMap->FindAndSet()找到可用的free block
# Q2
在dish.h中定義了以下的資訊:sector size為128Bytes，每一個track有32個sector，而每一個disk又有32個track，所以NachOS上模擬出來的硬碟大小為硬碟的大小為128*32*32 = 131072 Bytes
```javascript=
const int SectorSize = 128;     // number of bytes per disk sector
onst int SectorsPerTrack = 32; // number of sectors per disk track
const int NumTracks = 32;       // number of tracks per disk
```
# Q3
1.  一開始在FileSystem::FileSystem先create new directory和它的header
```javascript=
Directory *directory = new Directory(NumDirEntries);
FileHeader *dirHdr = new FileHeader;
```
2. 接下來將DirectorySector標記為已使用
```javascript=
freeMap->Mark(DirectorySector);
```
3. 將fileheader allocate在DirectorySector(1)上
```javascript=
ASSERT(dirHdr->Allocate(freeMap, DirectoryFileSize));
dirHdr->WriteBack(DirectorySector);
```
4.class Directory中:
* int tablesize: directory的entry數
* DirectoryEntry *table: DirectoryEntry 的陣列 pointer

5.而在class DirectoryEntry中:
* bool inUse: entry是否被使用
* int sector: disk裡sector的位置
* char name[FileNameMaxLen + 1]: Directory 的名字

# Q4
1.NachOS中inode是實作在Class FileHeaderg上，其中存有以下資訊
```javascript=
private:
    int numBytes;			// Number of bytes in the file
    int numSectors;			// Number of data sectors in the file
    int dataSectors[NumDirect];		// Disk sector numbers for each data 
```
2.以下為FileHeader::Allocate()，當創建檔案時會進入for迴圈執行BitMap::FindAndSet()進行allocation
```javascript=
bool
FileHeader::Allocate(PersistentBitmap *freeMap, int fileSize)
{ 
    numBytes = fileSize;
    numSectors  = divRoundUp(fileSize, SectorSize);
    if (freeMap->NumClear() < numSectors)
	return FALSE;		// not enough space

    for (int i = 0; i < numSectors; i++) {
	    dataSectors[i] = freeMap->FindAndSet();
	    // since we checked that there was enough free space,
	    // we expect this to succeed
	    ASSERT(dataSectors[i] >= 0);
    }
    return TRUE;
}
```
3.而在FindAndSet()內部會執行for迴圈在freemap中尋找可用的block，一旦找到就return目前的numBits
```javascript=
int 
Bitmap::FindAndSet() 
{
    for (int i = 0; i < numBits; i++) {
      if (!Test(i)) {
          Mark(i);
          return i;
      }
    }
    return -1;
}
```
4.在allocate的過程中可能會發生兩種情況:
第一種情況若空間是連續的話，則會像是contiguous allocation的模式

![](https://i.imgur.com/HZQ1XYa.png)

第二種情況若是free block的位置因為其他alloction而產生不連續的情況，則在FindAndSet()中會跳過被占用的block去分配其他free block，再Mark free block

![](https://i.imgur.com/HDyfdpT.png)

綜合以上兩種情況，這樣的allocation模式就會是fileheader去指向data sector
# Q5
1.在filehdr.h中，dataSectors用來儲存file位於disk上的sector位置
```javascript=
private:
    int numBytes;			// Number of bytes in the file
    int numSectors;			// Number of data sectors in the file
    int dataSectors[NumDirect];		// Disk sector numbers for each data 
```
2.我們可以看到在FileHeader::Allocate()中因為SectorSize為128bytes，而所以numSectors被限制在SectorSize/sizeof(int) = 128/4 = 32，所以一個file擁有的dataSectors陣列最多為32個，所以最大總容量為32* 128bytes(sector size) = 4KB
```javascript=
bool
FileHeader::Allocate(PersistentBitmap *freeMap, int fileSize)
{ 
    numBytes = fileSize;
    numSectors  = divRoundUp(fileSize, SectorSize);
    if (freeMap->NumClear() < numSectors)
	return FALSE;		// not enough space

    for (int i = 0; i < numSectors; i++) {
	    dataSectors[i] = freeMap->FindAndSet();
	    // since we checked that there was enough free space,
	    // we expect this to succeed
	    ASSERT(dataSectors[i] >= 0);
    }
    return TRUE;
}
```
