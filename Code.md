#include <windows.h>
#include <tchar.h>
#include <iostream>

void GetFileAttributesExample(const TCHAR* fileName) {
    WIN32_FILE_ATTRIBUTE_DATA fileAttributes;
    if (!GetFileAttributesEx(fileName, GetFileExInfoStandard, &fileAttributes)) {
        std::cerr << "Error retrieving file attributes.\n";
        return;
    }

    
    LARGE_INTEGER fileSize;
    fileSize.LowPart = fileAttributes.nFileSizeLow;
    fileSize.HighPart = fileAttributes.nFileSizeHigh;

    
    SYSTEMTIME creationTime, accessTime, writeTime;
    FileTimeToSystemTime(&fileAttributes.ftCreationTime, &creationTime);
    FileTimeToSystemTime(&fileAttributes.ftLastAccessTime, &accessTime);
    FileTimeToSystemTime(&fileAttributes.ftLastWriteTime, &writeTime);

    std::cout << "File size: " << fileSize.QuadPart << " bytes\n";
    std::cout << "Creation time: " << creationTime.wDay << "/" << creationTime.wMonth << "/" << creationTime.wYear << "\n";
    std::cout << "Last access time: " << accessTime.wDay << "/" << accessTime.wMonth << "/" << accessTime.wYear << "\n";
    std::cout << "Last write time: " << writeTime.wDay << "/" << writeTime.wMonth << "/" << writeTime.wYear << "\n";

    
    DWORD attributes = fileAttributes.dwFileAttributes;
    std::cout << "File attributes:\n";
    if (attributes & FILE_ATTRIBUTE_READONLY) std::cout << " - Read-only\n";
    if (attributes & FILE_ATTRIBUTE_HIDDEN) std::cout << " - Hidden\n";
    if (attributes & FILE_ATTRIBUTE_SYSTEM) std::cout << " - System\n";
    if (attributes & FILE_ATTRIBUTE_DIRECTORY) std::cout << " - Directory\n";

    
    PSID ownerSid = nullptr;
    PSECURITY_DESCRIPTOR securityDescriptor = nullptr;
    if (GetNamedSecurityInfo(fileName, SE_FILE_OBJECT, OWNER_SECURITY_INFORMATION,
                             &ownerSid, nullptr, nullptr, nullptr, &securityDescriptor) == ERROR_SUCCESS) {
        TCHAR ownerName[256];
        DWORD ownerNameSize = sizeof(ownerName) / sizeof(TCHAR);
        TCHAR domainName[256];
        DWORD domainNameSize = sizeof(domainName) / sizeof(TCHAR);
        SID_NAME_USE sidType;

        if (LookupAccountSid(nullptr, ownerSid, ownerName, &ownerNameSize, domainName, &domainNameSize, &sidType)) {
            std::wcout << L"Owner: " << ownerName << L"\n";
        } else {
            std::cerr << "Error looking up account SID.\n";
        }
        LocalFree(securityDescriptor);
    } else {
        std::cerr << "Error retrieving security information.\n";
    }
}
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
void ReadFileUsingC(const char* fileName) {
    FILE* file = fopen(fileName, "rb");
    if (!file) {
        std::cerr << "Error opening file.\n";
        return;
    }

    char buffer[4096];
    size_t bytesRead;
    while ((bytesRead = fread(buffer, 1, sizeof(buffer), file)) > 0) {
        
    }
    fclose(file);
}
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
void ReadFileUsingWinAPI(const TCHAR* fileName) {
    HANDLE file = CreateFile(fileName, GENERIC_READ, FILE_SHARE_READ, nullptr, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
    if (file == INVALID_HANDLE_VALUE) {
        std::cerr << "Error opening file.\n";
        return;
    }

    char buffer[4096];
    DWORD bytesRead;
    while (ReadFile(file, buffer, sizeof(buffer), &bytesRead, nullptr) && bytesRead > 0) {
    }
    CloseHandle(file);
}
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
void AsyncFileProcessing(const std::vector<const TCHAR*>& fileNames) {
    const int numFiles = fileNames.size();
    HANDLE files[numFiles];
    OVERLAPPED overlapped[numFiles] = {};
    char buffers[numFiles][4096] = {};

    for (int i = 0; i < numFiles; ++i) {
        files[i] = CreateFile(fileNames[i], GENERIC_READ, FILE_SHARE_READ, nullptr, OPEN_EXISTING, FILE_FLAG_OVERLAPPED, nullptr);
        if (files[i] == INVALID_HANDLE_VALUE) {
            std::cerr << "Error opening file: " << fileNames[i] << "\n";
            return;
        }

        overlapped[i].hEvent = CreateEvent(nullptr, TRUE, FALSE, nullptr);
    }

    for (int i = 0; i < numFiles; ++i) {
        DWORD bytesRead;
        ReadFile(files[i], buffers[i], sizeof(buffers[i]), &bytesRead, &overlapped[i]);
    }

    
    HANDLE events[numFiles];
    for (int i = 0; i < numFiles; ++i) {
        events[i] = overlapped[i].hEvent;
    }
    WaitForMultipleObjects(numFiles, events, TRUE, INFINITE);

    
    for (int i = 0; i < numFiles; ++i) {
        CloseHandle(files[i]);
        CloseHandle(overlapped[i].hEvent);
    }
}
