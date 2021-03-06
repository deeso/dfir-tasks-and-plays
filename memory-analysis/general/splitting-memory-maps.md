
### Overview

Task: Create separate files for each contiguous memory segment in a process dump
Description: Create separate files for each contiguous memory segment in a process dump

### Requirements
1. System with enough disk space to dump individual memory files
2. System with enough RAM to load memory (or modify script to be serial)
3. Volatility 3 is used to dump process memory and the memory map
4. Target directory for the segments

### Tasks
1. Extract each contiguous memory segment from a process dump

### Products
1. Contiguous memory segments from the process

### Post-activities
1. None

### Commands

Python script to perform the carving
```
PID = 'PID'
OUTPUT_DIR = 'memory_segments'
NUM_PROCS = 8
# assuming the following format for dumped process and map
MAP_FILENAME = "pid.{}.map".format(PID)
DMP_FILENAME = "pid.{}.dmp".format(PID)

import os
from multiprocessing import Pool

def read_map_file_entries(map_file):
    lines = open(map_file).readlines()
    mappings = [i.strip().split() for i in lines[4:]]
    current_offset = 0
    entries = []
    
    for ele in mappings:
        ele = [int(i, 16) for i in ele[:4]]
        vaddr, paddr, sz, dumpoffset = ele
        
        entries.append({
            'paddr': paddr, 'vaddr': vaddr, 'size':sz, 
            'offset':current_offset, 'dumpoffset':dumpoffset
        })
        current_offset += sz
    return entries


def build_memory_map(map_file):
    entries = read_map_file_entries(map_file)

    last = 0
    next = None
    mmaps = {}
    current = None
    last = None
    
    for ele in entries:
        vaddr = ele.get('vaddr')
        poffset = ele.get('offset')
        sz = ele.get('size')

        prot = 0
        if next is None:
            next = vaddr + sz
            mmaps[vaddr] = {'vaddr': vaddr, 'size':sz, 
                            "current_offset": poffset,
                            'entries': [ele],
                            # 'offset':[(dumpoffset, sz)], 
                            'vaddr_order':[vaddr], 'vaddr_size': {vaddr:sz},
                            'prot': prot}
            current = vaddr
            last = vaddr
            continue

        elif next != vaddr:
            next = vaddr + sz
            current = vaddr
            last = vaddr
            mmaps[vaddr] = {'vaddr': vaddr, 'size':sz, 
                            "current_offset": poffset,
                            'entries': [ele],
                            # 'offset':[(dumpoffset, sz)], 
                            'vaddr_order':[vaddr], 'vaddr_size': {vaddr:sz},
                            'prot': prot}

        next = vaddr + sz


        mmaps[current]['size'] += sz
        mmaps[current]['entries'].append(ele)
        # mmaps[current]['offset'].append((dumpoffset, sz))
        # mmaps[current]['vaddr_size'][vaddr] = sz
        mmaps[current]['vaddr_order'].append(vaddr)
    return mmaps

def write_data(vaddr, offset, size, output_dir, data_buffer):
    data = data_buffer[offset:offset+size]
    output = os.path.join(output_dir, "0x{:08x}-0x{:08x}.bin".format(vaddr, vaddr+offset))
    open(output, 'wb').write(data)


def mp_write_data(args):
    vaddr, offset, size, output_dir, data_buffer = args
    write_data(vaddr, offset, size, output_dir, data_buffer)


# produce the contiguous memory maps
mmaps = build_memory_map(MAP_FILENAME)

# load dump file
dmp = open(DMP_FILENAME, 'rb').read()

# dumped memory map offsets from the first memory address
# so actual offset is adjusted from this address
addrs = sorted(mmaps.keys())
base_addr = addrs[0]

# build the arguments for the multiprocessing
args = []
true_offset  = 0
for vaddr in addrs:
    mmap = mmaps[vaddr]
    # vaddr, offset, size, OUTPUT_DIR, data_buffer
    
    e = [
      vaddr, 
      mmap['current_offset'], 
      mmap['size'],
      OUTPUT_DIR,
      dmp 
    ]
    true_offset += mmap['size']
    args.append(e)

# create pool to parallelize the effort
pool = Pool(processes=NUM_PROCS)
r = pool.map_async(mp_write_data, args)
r.wait()
pool.terminate()
```
