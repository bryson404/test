-- ============================================
-- Kirtio SoftWorks
-- For CC: Tweaked (ComputerCraft)

-- ============================================

local JBOD = {
    version = "3.3",
    drives = {},
    debug = false
}

-- ============================================
-- DRIVE MANAGEMENT
-- ============================================

function JBOD:scanDrives()
    self.drives = {}
    local peripherals = peripheral.getNames()
    
    for _, name in ipairs(peripherals) do
        if peripheral.getType(name) == "drive" then
            local drive = peripheral.wrap(name)
            if drive.isDiskPresent and drive.isDiskPresent() then
                local mountPath = drive.getMountPath and drive.getMountPath()
                if mountPath then
                    table.insert(self.drives, {
                        name = name,
                        peripheral = drive,
                        mountPath = mountPath,
                        diskId = drive.getDiskID and drive.getDiskID() or 0,
                        fullPath = "/" .. mountPath
                    })
                end
            end
        end
    end
    
    table.sort(self.drives, function(a, b) return a.mountPath < b.mountPath end)
    return #self.drives
end

function JBOD:getPoolStats()
    local total, used, free = 0, 0, 0
    
    for i, drive in ipairs(self.drives) do
        local path = drive.fullPath
        if fs.isDir(path) then
            local cap = fs.getCapacity and fs.getCapacity(path) or 2000000
            local avail = fs.getFreeSpace(path)
            total = total + cap
            free = free + avail
        end
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
-- FILE OPERATIONS
-- ============================================

function JBOD:listFiles()
    local files = {}
    for driveIndex, drive in ipairs(self.drives) do
        local path = drive.fullPath
        if fs.isDir(path) then
            for _, name in ipairs(fs.list(path)) do
                local fp = path .. "/" .. name
                if not fs.isDir(fp) then
                    table.insert(files, {
                        name = name,
                        size = fs.getSize(fp),
                        drive = drive.mountPath
                    })
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
    if not f then return nil, "Not found" end
    for _, d in ipairs(self.drives) do
        if d.mountPath == f.drive then
            local h = fs.open(d.fullPath .. "/" .. name, "r")
            if h then
                local data = h.readAll()
                h.close()
                return data
            end
        end
    end
    return nil, "Read failed"
end

function JBOD:write(name, data)
    -- Check if exists (overwrite same drive)
    local existing = self:findFile(name)
    local targetDrive = nil
    
    if existing then
        for i, d in ipairs(self.drives) do
            if d.mountPath == existing.drive then
                targetDrive = i
                break
            end
        end
    else
        -- Find drive with most space
        local bestFree = -1
        for i, d in ipairs(self.drives) do
            local free = fs.getFreeSpace(d.fullPath)
            if free > bestFree then
                bestFree = free
                targetDrive = i
            end
        end
    end
    
    if not targetDrive then return false, "No drives" end
    
    local d = self.drives[targetDrive]
    if fs.getFreeSpace(d.fullPath) < #data then
        return false, "Not enough space"
    end
    
    local h = fs.open(d.fullPath .. "/" .. name, "w")
    if not h then return false, "Cannot create" end
    h.write(data)
    h.close()
    return true
end

function JBOD:delete(name)
    local f = self:findFile(name)
    if not f then return false, "Not found" end
    for _, d in ipairs(self.drives) do
        if d.mountPath == f.drive then
            fs.delete(d.fullPath .. "/" .. name)
            return true
        end
    end
    return false
end

-- ============================================
-- DISPLAY & UI
-- ============================================

function JBOD:formatBytes(b)
    if b >= 1000000 then return string.format("%.2f MB", b/1000000)
    elseif b >= 1000 then return string.format("%.2f KB", b/1000)
    else return b .. " B" end
end

function JBOD:drawBar(pct, w)
    local filled = math.floor((pct/100) * w)
    return "[" .. string.rep("=", filled) .. string.rep("-", w-filled) .. "]"
end

-- MAIN DISPLAY - Total Pool Only
function JBOD:drawMainScreen()
    term.clear()
    term.setCursorPos(1,1)
    
    local s = self:getPoolStats()
    
    print("===================================================")
    print("     JBOD POOL STORAGE")
    print("===================================================")
    print()
    print(string.format("  Drives: %d", s.drives))
    print()
    print(string.format("  %s %d%%", self:drawBar(s.percent, 30), s.percent))
    print()
    print(string.format("  Used:  %s", self:formatBytes(s.used)))
    print(string.format("  Free:  %s", self:formatBytes(s.free)))
    print(string.format("  Total: %s", self:formatBytes(s.total)))
    print()
    print("===================================================")
    print()
    
    -- Show files
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

-- DRIVE INFO SCREEN - Only when requested
function JBOD:showDriveInfo()
    term.clear()
    term.setCursorPos(1,1)
    
    print("===================================================")
    print("     INDIVIDUAL DRIVE INFORMATION")
    print("===================================================")
    print()
    
    if #self.drives == 0 then
        print("No drives detected!")
    else
        for i, d in ipairs(self.drives) do
            local cap = fs.getCapacity and fs.getCapacity(d.fullPath) or 2000000
            local free = fs.getFreeSpace(d.fullPath)
            local used = cap - free
            local pct = math.floor((used/cap)*100)
            
            print(string.format("[%d] %s (ID:%s)", i, d.mountPath, d.diskId))
            print(string.format("    %s %d%%", self:drawBar(pct, 20), pct))
            print(string.format("    %s / %s used, %s free", 
                self:formatBytes(used), self:formatBytes(cap), self:formatBytes(free)))
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
        
        term.setCursorPos(1, 20)
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
-- MAIN
-- ============================================

local args = {...}
if args[1] == "api" then
    JBOD:scanDrives()
    _G.JBOD = {
        list = function() return JBOD:listFiles() end,
        read = function(n) return JBOD:read(n) end,
        write = function(n,d) return JBOD:write(n,d) end,
        delete = function(n) return JBOD:delete(n) end,
        stats = function() return JBOD:getPoolStats() end,
        drives = function() return JBOD.drives end
    }
    print("JBOD API ready")
else
    JBOD:run()
end
