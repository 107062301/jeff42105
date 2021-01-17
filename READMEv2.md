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
1.在FileSystem中，我們一樣先去做traverse的動作，根據獲得info的資訊後找到sector的位置，再根據該sector的位置打開檔案並用我們在class FileSystem上新增的一個指標 OpenFile* currentOpen來表示現在開啟的檔案是哪一個
```javascript=
OpenFile*
FileSystem::Open(char* absolutePath)
{
    Directory* directory = new Directory(NumDirEntries);
    int sector;
    char* name = new char[256];
    strcpy(name, absolutePath);

    DEBUG(dbgFile, "Opening file" << name);
   
    directory->FetchFrom(directoryFile);
    // Pharse path
    TraverseFile* info = new TraverseFile();
    info = traverseFind(name);
    directory = info->directory;
    char* lastPath = info->lastpath;
    OpenFile* tmp_dir = info->tmp_dir;
   
    // file name is stored in lastPath
    sector = directory->Find(lastPath);
    if (sector >= 0)
        currentOpen = new OpenFile(sector);
    delete directory;
    if (tmp_dir != NULL)
        delete tmp_dir;
    delete[] name;
    return currentOpen;
}
```
## FileSystem::RecursiveList and FileSystem::List
1.在List或是RecursiveList的過程中，一樣也需要做path traverse，所以兩個函數在獲得info的資訊後會先確定該path存在，之後打開該path的directory，並呼叫Directory::RecursiveList(int layer)或是Directory::List()，以下只列出FileSystem::RecursiveList的code，FileSystem::List和它只差在path traverse完之後是呼叫Directory::RecursiveList(int layer)還是Directory::List()
```javascript=
void
FileSystem::RecursiveList(char* absolutePath)
{
    Directory* directory = new Directory(NumDirEntries);

    directory->FetchFrom(directoryFile);
    int sector;
    // MP4I

    char* name = new char[256];
    strcpy(name, absolutePath);
    // Pharse path
    TraverseFile* info = new TraverseFile();
    info = traverseFind(name);
    directory = info->directory;
    char* lastPath = info->lastpath;
    OpenFile* tmp_dir = info->tmp_dir;

    if (lastPath != NULL && (sector = directory->Find(lastPath)) != -1) {// case of /t0  lastPath:t0 Path:NULL but directory point to root
        if (tmp_dir != NULL) {
            delete tmp_dir;
            tmp_dir = NULL;
        }
        tmp_dir = new OpenFile(sector);
        directory->FetchFrom(tmp_dir);
    }

    directory->RecursiveList(1);
    if (tmp_dir != NULL)
        delete tmp_dir;
    delete[] name;
    delete directory;
}
```
## Directory::RecursiveList and Directory::List
1.在FileSystem中的FileSystem::RecursiveList / FileSystem::List會呼叫Directory::RecursiveList / Directory::List，首先以list來說，單純就是將現在的directory中所有元素都列出來，那我們根據該元素是否為directory在它們的名字前面加上一個前綴，[D]代表directory、[F]代表file
```javascript=
void
Directory::List()
{
    cout << "List Directory:" << endl;
    for (int i = 0; i < tableSize; i++) {
        if (table[i].inUse) {
            if (table[i].isdir) printf("[D]");
            else printf("[F]");
            printf("%s\n", table[i].name);
        }
    }
}
```
2.而RecursiveList()就需要用地回的方式印出現在的directory含有的元素，我們印出一個directory中的所有元素，並且對每一個directory都去將它底下的subdirectory打開並對該subdirectory做RecursiveList()，如此便可列出所有的directory和file，而排版的部分，一開始在FileSystem::RecursiveList中呼叫directory->RecursiveList(1)代表這是列出第一層directory，而當地一層的directory要去呼叫第二層時則會呼叫subdir->RecursiveList(2)，依此類推，並且根據現在所處的layer來決定要印出幾個空格，這裡以char space[1021]來儲存要印出的空格數量
```javascript=
void
Directory::RecursiveList(int layer) // modify
{
    char ch;
    char space[1021];
    if (layer == 1){
        cout << "Recursive List Directory:" << endl;
        space[0] = '\0';
    }
    else {
        for (int j = 0; j <= 4*(layer - 1)-1; j++) {
            space[j] = ' ';
        }
        space[4 * (layer - 1)] = '\0';
    }
    for (int i = 0; i < tableSize; i++) {
        if (table[i].inUse) {
            
            if (table[i].isdir) {
                ch = 'D';
                printf("%s[%c]: %s\n",space, ch, table[i].name);
            }
            else {
                ch = 'F';
                printf("%s[%c]: %s\n",space, ch, table[i].name);
            }
            
            if (table[i].isdir) {
                OpenFile* subdirfile = new OpenFile(table[i].sector);
                Directory* subdir = new Directory(NumDirEntries);
                subdir->FetchFrom(subdirfile);
                subdir->RecursiveList(layer+1);
                delete subdirfile;
                delete subdir;
            }
        }
    }
}
```
## FileSystem::Remove
1.在remove之前也事先traverse path，然後呼叫fileHdr->Deallocate(freeMap)來free data block，呼叫freeMap->Clear(sector)來移除header block，之後再將freemap writeback回disk上
```javascript=
bool
FileSystem::Remove(char* absolutePath)
{
    Directory* directory;
    PersistentBitmap* freeMap;
    FileHeader* fileHdr;
    int sector;
    char* name = new char[256];
    strcpy(name, absolutePath);

    directory = new Directory(NumDirEntries);
    directory->FetchFrom(directoryFile);

    // MP4I
    // Pharse path
    TraverseFile* info = new TraverseFile();
    info = traverseFind(name);
    directory = info->directory;
    char* lastPath = info->lastpath;
    OpenFile* tmp_dir = info->tmp_dir;

    // remove
    sector = directory->Find(lastPath);
    if (sector == -1) {
        delete directory;
        return FALSE;			 // file not found 
    }
    fileHdr = new FileHeader;
    fileHdr->FetchFrom(sector);

    freeMap = new PersistentBitmap(freeMapFile, NumSectors);

    fileHdr->Deallocate(freeMap);  		// remove data blocks
    freeMap->Clear(sector);			// remove header block
    directory->Remove(lastPath);

    freeMap->WriteBack(freeMapFile);		// flush to disk
    if (tmp_dir != NULL) {
        directory->WriteBack(tmp_dir); // flush to disk
        delete tmp_dir;
    }
    else {
        directory->WriteBack(directoryFile); // flush to disk
    }
    delete fileHdr;
    delete directory;
    delete freeMap;
    delete[] name;
    return TRUE;
}
```
## Support up to 64 files/subdirectories per directory
我們將 directory.cc/.h 和 filesys.cc/.h 的NumDirEntries由10改成64即可
## main.cc
1.main.cc為控制指令輸入的地方，所以我們在這裡做一些修改，我們用listDirectoryName來儲存輸入進list和recursive list的path的指標，當指令為-l時recursiveListFlag = false
```javascript=
else if (strcmp(argv[i], "-l") == 0)
        {
            // MP4 mod tag
            ASSERT(i + 1 < argc);
            listDirectoryName = argv[i + 1];
            dirListFlag = true;
            i++;
        }
```
2.當指令為-lr時recursiveListFlag = true
```javascript=
else if (strcmp(argv[i], "-lr") == 0)
        {
            // MP4 mod tag
            // recursive list
            ASSERT(i + 1 < argc);
            listDirectoryName = argv[i + 1];
            dirListFlag = true;
            recursiveListFlag = true;
            i++;
        }
```
3.我們用createDirectoryName來儲存輸入進CreateDir()的path的指標，當指令為-mkdir時mkdirFlag = true
```javascript=
else if (strcmp(argv[i], "-mkdir") == 0)
        {
            // MP4 mod tag
            ASSERT(i + 1 < argc);
            createDirectoryName = argv[i + 1];
            mkdirFlag = true;
            i++;
        }
```
4.我們用剛剛的mkdirFlag、recursiveListFlag和其他原本就有的變數來判斷要執行fileSystem的哪個function
```javascript=
if (removeFileName != NULL)
    {
        kernel->fileSystem->Remove(removeFileName);
    }
    if (copyUnixFileName != NULL && copyNachosFileName != NULL)
    {
        Copy(copyUnixFileName, copyNachosFileName);
    }
    if (dumpFlag)
    {
        kernel->fileSystem->Print();
    }
    if (dirListFlag)
    {
        if(recursiveListFlag) kernel->fileSystem->RecursiveList(listDirectoryName);
        else kernel->fileSystem->List(listDirectoryName);
        
    }
    if (mkdirFlag)
    {
        // MP4 mod tag
        CreateDirectory(createDirectoryName);
    }
    if (printFileName != NULL)
    {
        Print(printFileName);
    }
```
