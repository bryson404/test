8DiXrQet

    -- ============================================
    --  KIRTIO SOFTWORKS 
    -- Networking drives
    -- ============================================
     
    local JBOD = {
        version = "3.0",
        drives = {},
        mountPoint = "/jbod",
        debug = true
    }
     
    -- ============================================
    -- DRIVE DETECTION & MOUNT PATHS
    -- ============================================
     
    function JBOD:scanDrives()
        self.drives = {}
        local peripherals = peripheral.getNames()
     
        print("Scanning for drives...")
     
        for _, name in ipairs(peripherals) do
            local pType = peripheral.getType(name)
     
            -- Check if it's a drive peripheral
            if pType == "drive" then
                local drive = peripheral.wrap(name)
     
                -- Check if there's actually a disk/media in the drive
                local hasDisk = false
                if drive.isDiskPresent then
                    hasDisk = drive.isDiskPresent()
                end
     
                if hasDisk then
                    -- Get the mount path (e.g., "disk", "disk1")
                    local mountPath = nil
                    if drive.getMountPath then
                        mountPath = drive.getMountPath()
                    end
     
                    if mountPath then
                        -- Get disk info
                        local diskId = ""
                        if drive.getDiskID then
                            diskId = tostring(drive.getDiskID())
                        end
     
                        -- Check if it's a floppy or other media
                        local isFloppy = false
                        if drive.hasData then
                            isFloppy = drive.hasData()
                        end
     
                        table.insert(self.drives, {
                            name = name,
                            peripheral = drive,
                            mountPath = mountPath,
                            diskId = diskId,
                            isFloppy = isFloppy,
                            fullPath = "/" .. mountPath
                        })
     
                        if self.debug then
                            print("  Found: " .. name .. " -> " .. mountPath .. " (ID: " .. diskId .. ")")
                        end
                    else
                        if self.debug then
                            print("  Warning: " .. name .. " has disk but no mount path")
                        end
                    end
                else
                    if self.debug then
                        print("  " .. name .. " - no disk inserted")
                    end
                end
            end
        end
     
        -- Sort by mount path for consistent ordering
        table.sort(self.drives, function(a, b) return a.mountPath < b.mountPath end)
     
        print("Total drives mounted: " .. #self.drives)
        return #self.drives
    end
     
    -- ============================================
    -- FILE OPERATIONS (Using fs API on mount paths)
    -- ============================================
     
    function JBOD:listFiles()
        local allFiles = {}
     
        for driveIndex, drive in ipairs(self.drives) do
            local path = drive.fullPath
     
            -- Check if path exists and is directory
            if fs.isDir(path) then
                local files = fs.list(path)
                for _, filename in ipairs(files) do
                    local filePath = path .. "/" .. filename
                    local size = 0
                    if not fs.isDir(filePath) then
                        size = fs.getSize(filePath)
                    end
     
                    table.insert(allFiles, {
                        name = filename,
                        size = size,
                        driveIndex = driveIndex,
                        driveName = drive.name,
                        drivePath = drive.mountPath,
                        fullPath = filePath,
                        isDir = fs.isDir(filePath)
                    })
                end
            end
        end
     
        -- Sort by filename
        table.sort(allFiles, function(a, b) return a.name:lower() < b.name:lower() end)
     
        return allFiles
    end
     
    function JBOD:findFile(filename)
        local files = self:listFiles()
        for _, file in ipairs(files) do
            if file.name == filename then
                return file
            end
        end
        return nil
    end
     
    function JBOD:readFile(filename)
        local file = self:findFile(filename)
        if not file then
            return nil, "File not found: " .. filename
        end
     
        if file.isDir then
            return nil, "Is a directory"
        end
     
        -- Open and read using standard fs
        local handle = fs.open(file.fullPath, "r")
        if not handle then
            return nil, "Cannot open file"
        end
     
        local data = handle.readAll()
        handle.close()
     
        return data, file
    end
     
    function JBOD:writeFile(filename, data, targetDriveIndex)
        -- If file exists, overwrite on same drive
        local existing = self:findFile(filename)
        if existing then
            targetDriveIndex = existing.driveIndex
            if self.debug then
                print("Overwriting on drive " .. targetDriveIndex)
            end
        end
     
        -- If no target specified, find drive with most space
        if not targetDriveIndex then
            targetDriveIndex = self:findBestDrive(#data)
            if not targetDriveIndex then
                return false, "No drives with sufficient space"
            end
        end
     
        local drive = self.drives[targetDriveIndex]
        if not drive then
            return false, "Invalid drive index"
        end
     
        -- Check space
        local freeSpace = fs.getFreeSpace(drive.fullPath)
        if freeSpace < #data then
            return false, "Not enough space on " .. drive.mountPath .. " (need " .. #data .. ", have " .. freeSpace .. ")"
        end
     
        -- Write file
        local filePath = drive.fullPath .. "/" .. filename
        local handle = fs.open(filePath, "w")
        if not handle then
            return false, "Cannot create file"
        end
     
        handle.write(data)
        handle.close()
     
        if self.debug then
            print("Wrote " .. #data .. " bytes to " .. filePath)
        end
     
        return true, targetDriveIndex
    end
     
    function JBOD:deleteFile(filename)
        local file = self:findFile(filename)
        if not file then
            return false, "File not found"
        end
     
        fs.delete(file.fullPath)
        return true
    end
     
    function JBOD:moveFile(source, dest)
        local data, err = self:readFile(source)
        if not data then
            return false, err
        end
     
        local ok, err2 = self:writeFile(dest, data)
        if not ok then
            return false, err2
        end
     
        self:deleteFile(source)
        return true
    end
     
    -- ============================================
    -- SPACE MANAGEMENT
    -- ============================================
     
    function JBOD:getDriveStats(driveIndex)
        local drive = self.drives[driveIndex]
        if not drive then return nil end
     
        local total = fs.getCapacity and fs.getCapacity(drive.fullPath) or 2000000 -- Default 2MB for floppies
        local free = fs.getFreeSpace(drive.fullPath)
        local used = total - free
     
        return {
            total = total,
            free = free,
            used = used,
            percent = math.floor((used / total) * 100)
        }
    end
     
    function JBOD:findBestDrive(sizeNeeded)
        local bestIndex = nil
        local mostFree = -1
     
        for i, drive in ipairs(self.drives) do
            local free = fs.getFreeSpace(drive.fullPath)
            if free >= sizeNeeded and free > mostFree then
                mostFree = free
                bestIndex = i
            end
        end
     
        return bestIndex
    end
     
    -- ============================================
    -- USER INTERFACE
    -- ============================================
     
    function JBOD:formatBytes(bytes)
        if bytes >= 1000000000 then
            return string.format("%.2f GB", bytes / 1000000000)
        elseif bytes >= 1000000 then
            return string.format("%.2f MB", bytes / 1000000)
        elseif bytes >= 1000 then
            return string.format("%.2f KB", bytes / 1000)
        else
            return bytes .. " B"
        end
    end
     
    function JBOD:drawBar(percent, width)
        local filled = math.floor((percent / 100) * width)
        filled = math.max(0, math.min(filled, width))
        local bar = string.rep("█", filled) .. string.rep("░", width - filled)
        return "[" .. bar .. "]"
    end
     
    function JBOD:drawInterface()
        term.clear()
        term.setCursorPos(1, 1)
     
        print("====================================================")
        print("|     Kritio Softworks Drive Manager    "          |")
        print("====================================================")
        print()
     
        -- Drive Status
        print("DRIVE STATUS:")
        print(string.rep("-", 50))
     
        local totalSpace = 0
        local totalFree = 0
     
        for i, drive in ipairs(self.drives) do
            local stats = self:getDriveStats(i)
            if stats then
                totalSpace = totalSpace + stats.total
                totalFree = totalFree + stats.free
     
                print(string.format("[%d] %-10s %s", i, drive.mountPath, drive.name))
                print(string.format("    ID:%s  %s %d%% (%s free)", 
                    drive.diskId, 
                    self:drawBar(stats.percent, 15),
                    stats.percent,
                    self:formatBytes(stats.free)
                ))
            end
        end
     
        if #self.drives == 0 then
            print("  No drives detected!")
            print("  Insert disks and press S to scan")
        end
     
        print(string.rep("-", 50))
        local totalUsed = totalSpace - totalFree
        local totalPercent = totalSpace > 0 and math.floor((totalUsed / totalSpace) * 100) or 0
        print(string.format("TOTAL: %s used / %s total (%d%%)", 
            self:formatBytes(totalUsed),
            self:formatBytes(totalSpace),
            totalPercent
        ))
     
        -- File List Preview
        print()
        print("FILES:")
        print(string.rep("-", 50))
     
        local files = self:listFiles()
        if #files == 0 then
            print("  (empty)")
        else
            for i = 1, math.min(8, #files) do
                local f = files[i]
                local sizeStr = f.isDir and "<DIR>" or self:formatBytes(f.size)
                print(string.format("  %-20s %10s  [%s]", 
                    f.name:sub(1,20), 
                    sizeStr,
                    f.drivePath
                ))
            end
            if #files > 8 then
                print("  ... and " .. (#files - 8) .. " more")
            end
        end
     
        print()
        print("Commands: [L]ist  [R]ead  [W]rite  [D]elete  [M]ove  [S]can  [Q]uit")
    end
     
    function JBOD:interactiveShell()
        self:scanDrives()
     
        while true do
            self:drawInterface()
     
            term.setCursorPos(1, 22)
            write("> ")
     
            local input = read():lower()
            local cmd = input:sub(1, 1)
     
            if cmd == "q" then
                print("Goodbye!")
                break
     
            elseif cmd == "s" then
                print("Scanning...")
                self:scanDrives()
                sleep(0.5)
     
            elseif cmd == "l" then
                print("\nAll Files:")
                local files = self:listFiles()
                for _, f in ipairs(files) do
                    local sizeStr = f.isDir and "<DIR>" or self:formatBytes(f.size)
                    print(string.format("  %-25s %10s  %s", 
                        f.name, sizeStr, f.drivePath
                    ))
                end
                print("\nPress Enter to continue...")
                read()
     
            elseif cmd == "r" then
                write("Filename to read: ")
                local name = read()
                write("Save as (or 'print'): ")
                local dest = read()
     
                local data, info = self:readFile(name)
                if data then
                    if dest == "print" then
                        print("\n--- Content ---")
                        print(data)
                        print("--- End ---")
                    else
                        local f = fs.open(dest, "w")
                        if f then
                            f.write(data)
                            f.close()
                            print("Saved to " .. dest)
                        else
                            print("Failed to save locally")
                        end
                    end
                else
                    print("Error: " .. tostring(info))
                end
                sleep(1)
     
            elseif cmd == "w" then
                write("Local file to upload: ")
                local localFile = read()
     
                if not fs.exists(localFile) then
                    print("File not found: " .. localFile)
                else
                    -- Check if it's a file, not directory
                    if fs.isDir(localFile) then
                        print("Cannot upload directories yet")
                    else
                        local f = fs.open(localFile, "r")
                        local data = f.readAll()
                        f.close()
     
                        write("Save as [" .. fs.getName(localFile) .. "]: ")
                        local jbodName = read()
                        if jbodName == "" then
                            jbodName = fs.getName(localFile)
                        end
     
                        print("Uploading " .. self:formatBytes(#data) .. "...")
                        local ok, err = self:writeFile(jbodName, data)
                        if ok then
                            print("Success! Saved to drive " .. err) -- err contains drive index on success
                        else
                            print("Failed: " .. tostring(err))
                        end
                    end
                end
                sleep(1)
     
            elseif cmd == "d" then
                write("Filename to delete: ")
                local name = read()
                write("Confirm delete '" .. name .. "'? (yes/no): ")
                if read():lower() == "yes" then
                    local ok, err = self:deleteFile(name)
                    if ok then
                        print("Deleted")
                    else
                        print("Failed: " .. tostring(err))
                    end
                else
                    print("Cancelled")
                end
                sleep(1)
     
            elseif cmd == "m" then
                write("Source: ")
                local src = read()
                write("Destination: ")
                local dst = read()
     
                local ok, err = self:moveFile(src, dst)
                if ok then
                    print("Moved successfully")
                else
                    print("Failed: " .. tostring(err))
                end
                sleep(1)
            end
        end
    end
     
    -- ============================================
    -- EXPORT API
    -- ============================================
     
    function JBOD:exportAPI()
        _G.JBOD_API = {
            list = function() return self:listFiles() end,
            read = function(name) return self:readFile(name) end,
            write = function(name, data) return self:writeFile(name, data) end,
            delete = function(name) return self:deleteFile(name) end,
            move = function(src, dst) return self:moveFile(src, dst) end,
            scan = function() return self:scanDrives() end,
            getDrives = function() return self.drives end,
            getStats = function() 
                local stats = {}
                for i = 1, #self.drives do
                    stats[i] = self:getDriveStats(i)
                end
                return stats
            end
        }
    end
     
    -- ============================================
    -- MAIN
    -- ============================================
     
    local function main(args)
        print("JBOD Network Storage v" .. JBOD.version)
     
        if args[1] == "daemon" then
            JBOD:scanDrives()
            JBOD:exportAPI()
            print("Daemon running. API available as JBOD_API")
            while true do
                sleep(60)
                JBOD:scanDrives()
            end
        elseif args[1] == "api" then
            JBOD:scanDrives()
            JBOD:exportAPI()
            print("API exported")
        else
            JBOD:interactiveShell()
        end
    end
     
    main({...})

