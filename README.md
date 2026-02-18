-- ============================================
-- KIRITO SOFTWORKS
-- For CC: Tweaked (ComputerCraft)
-- ============================================

local JBOD = {
    version = "3.4",
    drives = {},
    debug = false
}

-- ============================================
-- SUPPORTED PERIPHERAL TYPES
-- ============================================

-- List of peripheral types that can act as storage
JBOD.VALID_TYPES = {
    "drive",           -- Standard CC:Tweaked disk drive
    "disk_drive",      -- Alternative naming
    "diskraid",        -- More Peripherals Disk Raid
    "disk_raid",       -- Alternative naming
    "advanced_disk_raid", -- More Peripherals Advanced Disk Raid
    "storage",         -- Generic storage peripheral
    "drive_bay",       -- Possible modded drive bays
    "media_drive",     -- Other media drives
}

function JBOD:isValidStorageType(peripheralType)
    if not peripheralType then return false end
    peripheralType = peripheralType:lower()
    
    for _, validType in ipairs(self.VALID_TYPES) do
        if peripheralType:find(validType) then
            return true
        end
    end
    return false
end

-- ============================================
-- DRIVE DETECTION
-- ============================================

function JBOD:scanDrives()
    self.drives = {}
    local peripherals = peripheral.getNames()
    
    print("Scanning for storage devices...")
    
    for _, name in ipairs(peripherals) do
        local pType = peripheral.getType(name)
        
        if self:isValidStorageType(pType) then
            local drive = peripheral.wrap(name)
            
            -- Check if disk/media is present
            local hasMedia = false
            
            -- Try different methods to check for media
            if drive.isDiskPresent then
                hasMedia = drive.isDiskPresent()
            elseif drive.hasDisk then
                hasMedia = drive.hasDisk()
            elseif drive.isMediaPresent then
                hasMedia = drive.isMediaPresent()
            elseif drive.getDiskID then
                -- If it has getDiskID, try calling it
                local ok, result = pcall(function() return drive.getDiskID() end)
                hasMedia = ok and result ~= nil
            else
                -- Assume present if we can't check (some raid blocks don't report this)
                hasMedia = true
            end
            
            if hasMedia then
                -- Get mount path
                local mountPath = nil
                
                if drive.getMountPath then
                    mountPath = drive.getMountPath()
                end
                
                -- For raid blocks or drives without standard mount paths
                -- we'll check if they have direct fs methods or need special handling
                if not mountPath and drive.list then
                    -- Some raid blocks have direct fs methods
                    mountPath = name  -- Use peripheral name as identifier
                end
                
                if mountPath or drive.list then
                    local diskId = "N/A"
                    if drive.getDiskID then
                        local ok, id = pcall(function() return drive.getDiskID() end)
                        if ok then diskId = tostring(id) end
                    elseif drive.getID then
                        local ok, id = pcall(function() return drive.getID() end)
                        if ok then diskId = tostring(id) end
                    end
                    
                    table.insert(self.drives, {
                        name = name,
                        peripheral = drive,
                        pType = pType,
                        mountPath = mountPath or name,
                        diskId = diskId,
                        fullPath = mountPath and ("/" .. mountPath) or nil,
                        isRaid = pType:find("raid") ~= nil
                    })
                    
                    if self.debug then
                        print("  Found: " .. name .. " [" .. pType .. "] -> " .. (mountPath or name))
                    end
                end
            else
                if self.debug then
                    print("  " .. name .. " [" .. pType .. "] - no media")
                end
            end
        end
    end
    
    table.sort(self.drives, function(a, b) return a.name < b.name end)
    print("Total storage devices: " .. #self.drives)
    return #self.drives
end

-- ============================================
-- FILE OPERATIONS
-- ============================================

function JBOD:listFiles()
    local files = {}
    
    for driveIndex, drive in ipairs(self.drives) do
        local p = drive.peripheral
        
        -- Method 1: Standard mount path (regular disk drives)
        if drive.fullPath and fs.isDir(drive.fullPath) then
            for _, name in ipairs(fs.list(drive.fullPath)) do
                local fp = drive.fullPath .. "/" .. name
                if not fs.isDir(fp) then
                    table.insert(files, {
                        name = name,
                        size = fs.getSize(fp),
                        driveIndex = driveIndex,
                        drive = drive.name,
                        driveType = drive.pType
                    })
                end
            end
            
        -- Method 2: Direct peripheral methods (raid blocks)
        elseif p.list then
            local ok, items = pcall(function() return p.list("/") end)
            if ok and items then
                for name, size in pairs(items) do
                    if type(size) == "number" then
                        table.insert(files, {
                            name = name,
                            size = size,
                            driveIndex = driveIndex,
                            drive = drive.name,
                            driveType = drive.pType
                        })
                    end
                end
            end
        end
    end
    
    table.sort(files, function(a, b) return a.name:lower() < b.name:lower() end)
    return files
end

function JBOD:findFile(name)
    for _, f in ipairs(self:listFiles()) do
        if f.name == name then return f end
    end
    return nil
end

function JBOD:read(name)
    local f = self:findFile(name)
    if not f then return nil, "File not found" end
    
    local drive = self.drives[f.driveIndex]
    local p = drive.peripheral
    
    -- Try method 1: Standard fs
    if drive.fullPath then
        local path = drive.fullPath .. "/" .. name
        local h = fs.open(path, "r")
        if h then
            local data = h.readAll()
            h.close()
            return data
        end
    end
    
    -- Try method 2: Direct read
    if p.read then
        local ok, data = pcall(function() 
            local h = p.read(name)
            if h then
                local d = h.readAll()
                h.close()
                return d
            end
        end)
        if ok and data then return data end
    end
    
    return nil, "Read failed"
end

function JBOD:write(name, data)
    local existing = self:findFile(name)
    local targetDrive = nil
    
    -- Overwrite existing
    if existing then
        targetDrive = existing.driveIndex
    else
        -- Find drive with most space
        local bestFree = -1
        for i, d in ipairs(self.drives) do
            local free = self:getDriveFreeSpace(i)
            if free > bestFree then
                bestFree = free
                targetDrive = i
            end
        end
    end
    
    if not targetDrive then return false, "No drives available" end
    
    local drive = self.drives[targetDrive]
    
    -- Check space
    if self:getDriveFreeSpace(targetDrive) < #data then
        return false, "Not enough space"
    end
    
    -- Try method 1: Standard fs
    if drive.fullPath then
        local path = drive.fullPath .. "/" .. name
        local h = fs.open(path, "w")
        if h then
            h.write(data)
            h.close()
            return true
        end
    end
    
    -- Try method 2: Direct write
    if drive.peripheral.write then
        local ok = pcall(function()
            local h = drive.peripheral.write(name)
            if h then
                h.write(data)
                h.close()
                return true
            end
        end)
        if ok then return true end
    end
    
    return false, "Write failed"
end

function JBOD:delete(name)
    local f = self:findFile(name)
    if not f then return false, "Not found" end
    
    local drive = self.drives[f.driveIndex]
    
    -- Method 1: Standard fs
    if drive.fullPath then
        fs.delete(drive.fullPath .. "/" .. name)
        return true
    end
    
    -- Method 2: Direct delete
    if drive.peripheral.delete then
        local ok = pcall(function() drive.peripheral.delete(name) end)
        return ok
    end
    
    return false
end

-- ============================================
-- SPACE CALCULATION
-- ============================================

function JBOD:getDriveFreeSpace(index)
    local drive = self.drives[index]
    if not drive then return 0 end
    
    -- Method 1: fs.getFreeSpace
    if drive.fullPath then
        return fs.getFreeSpace(drive.fullPath)
    end
    
    -- Method 2: peripheral method
    if drive.peripheral.getFreeSpace then
        local ok, free = pcall(function() return drive.peripheral.getFreeSpace() end)
        if ok then return free end
    end
    
    -- Method 3: Calculate from capacity - used
    if drive.peripheral.getCapacity and drive.peripheral.getUsedSpace then
        local ok1, cap = pcall(function() return drive.peripheral.getCapacity() end)
        local ok2, used = pcall(function() return drive.peripheral.getUsedSpace() end)
        if ok1 and ok2 then return cap - used end
    end
    
    return 0
end

function JBOD:getDriveTotalSpace(index)
    local drive = self.drives[index]
    if not drive then return 0 end
    
    if drive.peripheral.getCapacity then
        local ok, cap = pcall(function() return drive.peripheral.getCapacity() end)
        if ok then return cap end
    end
    
    if fs.getCapacity and drive.fullPath then
        local ok, cap = pcall(function() return fs.getCapacity(drive.fullPath) end)
        if ok then return cap end
    end
    
    -- Default estimates
    if drive.isRaid then return 10000000 end -- 10MB estimate for raid
    return 2000000 -- 2MB default for floppy
end

function JBOD:getPoolStats()
    local total, used, free = 0, 0, 0
    
    for i = 1, #self.drives do
        local cap = self:getDriveTotalSpace(i)
        local avail = self:getDriveFreeSpace(i)
        total = total + cap
        free = free + avail
    end
    
    used = total - free
    local percent = total > 0 and math.floor((used / total) * 100) or 0
    
    return {
        total = total,
        used = used,
        free = free,
        percent = percent,
        drives = #self.drives
    }
end

-- ============================================
-- USER INTERFACE - KIRITO SOFTWORKS
-- ============================================

function JBOD:formatBytes(b)
    if b >= 1000000000 then return string.format("%.2f GB", b/1000000000)
    elseif b >= 1000000 then return string.format("%.2f MB", b/1000000)
    elseif b >= 1000 then return string.format("%.2f KB", b/1000)
    else return b .. " B" end
end

function JBOD:drawBar(pct, w)
    local filled = math.floor((pct/100) * w)
    return "[" .. string.rep("=", filled) .. string.rep("-", w-filled) .. "]"
end

function JBOD:drawHeader()
    print("===================================================")
    print("           KIRITO SOFTWORKS PRESENTS")
    print("===================================================")
    print()
end

function JBOD:drawMainScreen()
    term.clear()
    term.setCursorPos(1,1)
    
    self:drawHeader()
    
    local s = self:getPoolStats()
    
    print("JBOD NETWORK STORAGE POOL")
    print("---------------------------------------------------")
    print(string.format("  Storage Devices: %d", s.drives))
    print()
    print(string.format("  %s %d%%", self:drawBar(s.percent, 30), s.percent))
    print()
    print(string.format("  Used:  %s", self:formatBytes(s.used)))
    print(string.format("  Free:  %s", self:formatBytes(s.free)))
    print(string.format("  Total: %s", self:formatBytes(s.total)))
    print("---------------------------------------------------")
    print()
    
    local files = self:listFiles()
    print("FILES (" .. #files .. " total):")
    if #files == 0 then
        print("  (pool empty)")
    else
        for i = 1, math.min(8, #files) do
            local f = files[i]
            print(string.format("  %-20s %s", f.name:sub(1,20), self:formatBytes(f.size)))
        end
        if #files > 8 then print("  ... +" .. (#files-8) .. " more") end
    end
    
    print()
    print("Commands: [L]ist  [R]ead  [W]rite  [D]elete  [I]nfo  [Q]uit")
end

-- Drive info screen (only when requested)
function JBOD:showDriveInfo()
    term.clear()
    term.setCursorPos(1,1)
    
    self:drawHeader()
    print("INDIVIDUAL STORAGE DEVICES")
    print("===================================================")
    print()
    
    if #self.drives == 0 then
        print("No storage devices detected!")
    else
        for i, d in ipairs(self.drives) do
            local total = self:getDriveTotalSpace(i)
            local free = self:getDriveFreeSpace(i)
            local used = total - free
            local pct = math.floor((used/total)*100)
            
            print(string.format("[%d] %s", i, d.name))
            print(string.format("    Type: %s%s", 
                d.pType, 
                d.isRaid and " [RAID]" or ""
            ))
            print(string.format("    ID: %s", d.diskId))
            print(string.format("    %s %d%%", self:drawBar(pct, 20), pct))
            print(string.format("    %s / %s used", self:formatBytes(used), self:formatBytes(total)))
            print(string.format("    %s free", self:formatBytes(free)))
            print()
        end
    end
    
    print("===================================================")
    print("Press Enter to return...")
    read()
end

-- ============================================
-- INTERACTIVE SHELL
-- ============================================

function JBOD:run()
    self:scanDrives()
    
    while true do
        self:drawMainScreen()
        
        term.setCursorPos(1, 22)
        write("> ")
        local cmd = read():lower():sub(1,1)
        
        if cmd == "q" then
            print("Goodbye!")
            break
            
        elseif cmd == "i" then
            self:showDriveInfo()
            
        elseif cmd == "l" then
            print("\nAll files:")
            for _, f in ipairs(self:listFiles()) do
                print(string.format("  %-25s %10s  [%s]", f.name, self:formatBytes(f.size), f.drive))
            end
            print("\nPress Enter...")
            read()
            
        elseif cmd == "r" then
            write("File to read: ")
            local name = read()
            write("Save as: ")
            local dest = read()
            
            local data, err = self:read(name)
            if data then
                local h = fs.open(dest, "w")
                if h then
                    h.write(data)
                    h.close()
                    print("Saved to " .. dest)
                else
                    print("Save failed")
                end
            else
                print("Error: " .. tostring(err))
            end
            sleep(1)
            
        elseif cmd == "w" then
            write("Local file: ")
            local src = read()
            if not fs.exists(src) or fs.isDir(src) then
                print("Not found or is directory")
            else
                local h = fs.open(src, "r")
                local data = h.readAll()
                h.close()
                
                write("Save as [" .. fs.getName(src) .. "]: ")
                local name = read()
                if name == "" then name = fs.getName(src) end
                
                local ok, err = self:write(name, data)
                print(ok and "Saved!" or "Failed: " .. tostring(err))
            end
            sleep(1)
            
        elseif cmd == "d" then
            write("File to delete: ")
            local name = read()
            write("Confirm (yes): ")
            if read():lower() == "yes" then
                local ok = self:delete(name)
                print(ok and "Deleted" or "Failed")
            else
                print("Cancelled")
            end
            sleep(1)
        end
    end
end

-- ============================================
-- API EXPORT
-- ============================================

function JBOD:exportAPI()
    _G.JBOD = {
        list = function() return self:listFiles() end,
        read = function(n) return self:read(n) end,
        write = function(n,d) return self:write(n,d) end,
        delete = function(n) return self:delete(n) end,
        stats = function() return self:getPoolStats() end,
        drives = function() return self.drives end,
        scan = function() return self:scanDrives() end
    }
end

-- ============================================
-- MAIN
-- ============================================

local args = {...}
if args[1] == "api" then
    JBOD:scanDrives()
    JBOD:exportAPI()
    print("KIRITO SOFTWORKS JBOD API ready")
elseif args[1] == "daemon" then
    JBOD:scanDrives()
    JBOD:exportAPI()
    print("KIRITO SOFTWORKS JBOD Daemon running")
    while true do
        sleep(60)
        JBOD:scanDrives()
    end
else
    JBOD:run()
end
