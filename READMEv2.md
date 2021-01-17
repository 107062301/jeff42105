# Part3-Modify the file system code to support subdirectory
## Directory::Add
1.  首先我們要在class DirectoryEntry中多增加一個變數isdir因為在一個directory中除了file之外還有可能會是subdiirectory，所以需要用isdir來判斷directory底下的node是否為file
```javascript=
class DirectoryEntry {
  public:
    bool inUse;				// Is this directory entry in use?
    bool isdir; //modify
    int sector;				// Location on disk to find the 
					//   FileHeader for this file 
    char name[FileNameMaxLen + 1];	// Text name for file, with +1 for 
					// the trailing '\0'
};
```
2.當我們要add新的file時，根據傳入的isdir來決定add的是file還是directory，並記錄在table[i].isdir中
```javascript=
bool
Directory::Add(char *name, int newSector, bool isdir)
{ 
    if (FindIndex(name) != -1)
	return FALSE;

    for (int i = 0; i < tableSize; i++) {
        if (!table[i].inUse) {
            table[i].inUse = TRUE;
            strncpy(table[i].name, name, FileNameMaxLen);
            table[i].sector = newSector;

            table[i].isdir = (isdir) ? true : false; //modify
            
            return TRUE;
        }
    }
    return FALSE;	// no space.  Fix when we have extensible files.
}
```
## 建立class TraverseFile & 在class FileSystem中加入函數TraverseFile* traverseFind(char* name);
1.我們先建立新的class TraverseFile，用來儲存等等要用在traverseFind()的回傳值上
* directory: ⽬前所處的 directory。當 filename 指向 file 時，此變數指向其隸屬的 directory ，當
filename 指向 directory 時，此變數也指向它。
* tmp_dir: 暫存目前開啟的directory
* lastpath: 最後的 filename 或 directory name。
```javascript=
class TraverseFile {
public:
    TraverseFile() {
        directory = new Directory(NumDirEntries);
    }
    ~TraverseFile() {
        delete directory;
    }
    Directory* directory;
    OpenFile* tmp_dir;
    char *lastpath;
};
```
2.因為在class FileSystem中幾乎所有的操作都需要根據path找到相關的位置資訊，所以我們以class TraverseFile來儲存需要回傳的各項數值，並且在traverseFind()return前賦值，再將此函數使用到各個file system上的操作
* 其中while loop就是用來尋找該路徑的lastpath、tmp_dir和directory變數，透過strtok()來逐步拆解path達到traverse的效果，直到traverse到的file或directory不存在即停止
```javascript=
TraverseFile* 
FileSystem::traverseFind(char* name) {

    TraverseFile* info = new TraverseFile();
    Directory* directory = new Directory(NumDirEntries);
    directory->FetchFrom(directoryFile);
    OpenFile* tmp_dir = NULL;
    int sector;
    char* path = strtok(name, "/");
    char* lastPath = path;
    path = strtok(NULL, "/");

    while (path != NULL) {// traverse through directory
        if ((sector = directory->Find(lastPath)) == -1)// name not found in directory
            break;

        // Open the directory and go through
        if (tmp_dir != NULL) {
            delete tmp_dir;
            tmp_dir = NULL;
        }
        tmp_dir = new OpenFile(sector);
        directory->FetchFrom(tmp_dir);
        lastPath = path;
        path = strtok(NULL, "/");
    }

    info->tmp_dir = tmp_dir;
    info->directory = directory;
    info->lastpath = lastPath;

    return info;
}
```
## FileSystem::CreateDirectory and FileSystem::Create
1. 接下來我們針對FileSystem::Create做修改，為了要能支援subdirectory，我們要能夠根據傳入的path來建立file，所以我們用name這個變數來做一個path的copy並且傳入traverseFind(name)中來獲得參數
```javascript=
Directory* directory;
directory = new Directory(NumDirEntries);
directory->FetchFrom(directoryFile);
TraverseFile* info = new TraverseFile();
info = traverseFind(name);
directory = info->directory;
char* lastPath = info->lastpath;
OpenFile* tmp_dir = info->tmp_dir;  
```
2. 接下來若成功找到file path，則我們再allocate空間給file header，並且判斷directory、free map、disk是否有足夠空間，若順利執行，則根據我們從info這個指標的指到的tmp_dir是否為空來決定要writeback的directory，接下來delete剛剛allocate的指標來釋放空間，並return success來回傳成功與否
```javascript=
if (directory->Find(lastPath) != -1)
        success = FALSE;			// file is already in directory
    else {
        freeMap = new PersistentBitmap(freeMapFile, NumSectors);
        sector = freeMap->FindAndSet();	// find a sector to hold the file header
        if (sector == -1) {
            success = FALSE;		// no free block for file header 
            cout << "No free space\n";
        }
        else if (!directory->Add(lastPath, sector, 0)) {
            success = FALSE;	// no space in directory
            cout << "Can't create in directory\n";
        }
        else {
            hdr = new FileHeader;
            if (!hdr->Allocate(freeMap, initialSize))
                success = FALSE;	// no space on disk for data
            else {
                success = TRUE;
                // everthing worked, flush all changes back to disk
                hdr->WriteBack(sector);
                if (tmp_dir == NULL)
                    directory->WriteBack(directoryFile);
                else
                    directory->WriteBack(tmp_dir);
                freeMap->WriteBack(freeMapFile);
            }
            delete hdr;
        }
        delete freeMap;
    }
    delete directory;
    delete[] name;
    return success;
```
3.FileSystem::CreateDir大致相似，較不同的地方在於傳入* Add(char *name, int newSector, bool isdir);* 中的isdir為true代表這是一個directory，還有我們要將新增的new Directory writeback回disk中
```javascript=
if (directory->Find(lastPath) != -1)
        success = FALSE;			// file is already in directory
    else {
        Directory* new_dir = new Directory(NumDirEntries);
        freeMap = new PersistentBitmap(freeMapFile, NumSectors);
        sector = freeMap->FindAndSet();	// find a sector to hold the file header
        if (sector == -1)
            success = FALSE;		// no free block for file header 
        else if (!directory->Add(lastPath, sector, 1))
            success = FALSE;	// no space in directory
        else {
            hdr = new FileHeader;
            if (!hdr->Allocate(freeMap, DirectoryFileSize))
                success = FALSE;	// no space on disk for data
            else {
                success = TRUE;
                // everthing worked, flush all changes back to disk
                hdr->WriteBack(sector);
                if (tmp_dir == NULL)
                    directory->WriteBack(directoryFile);
                else
                    directory->WriteBack(tmp_dir);
                freeMap->WriteBack(freeMapFile);
                OpenFile* newDirFile = new OpenFile(sector);
                new_dir->WriteBack(newDirFile);
                delete newDirFile;
            }
            delete hdr;
        }
        delete new_dir;
        delete freeMap;
    }
```
## FileSystem::Open

4.freemap 中由1和0組成，若為1代表已被占用若為0代表為未使用，可以透過freeMap->FindAndSet()找到可用的free block
# Q2
在dish.h中定義了以下的資訊:sector size為128Bytes，每一個track有32個sector，而每一個disk又有32個track，所以NachOS上模擬出來的硬碟大小為硬碟的大小為128*32*32 = 131072 Bytes
```javascript=
const int SectorSize = 128;     // number of bytes per disk sector
onst int SectorsPerTrack = 32; // number of sectors per disk track
const int NumTracks = 32;       // number of tracks per disk
```
```javascript=

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
