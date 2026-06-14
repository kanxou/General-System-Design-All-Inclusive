DBMS = Disk
RAM = DataStructure

Sector = Cake piece like section of disk
Track = Circular Tracks on the disk 

Block = TrackId + SectorId

Block = Lowest level at which the data can be read from the disk, even if we want 1 byte of data , we have to pull entire block 4Kb (earlier 512 and used in this example from Abdul Bari video)
DB Page =  Smallest chunk of data that the db reads into its cache (pool buffer) (typically 8 or 16Kb) 
Single DB Page spans across multiple blocks of disk

Rotation of disk helps to navigate through sectors
The pin moves forward backward helping select the track

In this way we are able to locate a block , Block size = 512Byte 
**Offset
The distance (measured in bytes) from the absolute beginning of a block to the specific piece of data you want to read.
If a block starts at byte 0 and your target record is at byte 100, your offset is 100**

If we have a table employee , with fields like id, name, address etc lets say the size of each row is 128 Bytes
Each block can hold = 512/128 = 4 Rows i.e 1 Row = 1/4th block
No of blocks required for 100 rows i.e 25 blokcs

This is too much to check for a single row, how about indexing??

We make a new table where each key points to the location of the exact row. It has two fields i.e key and second location, lets say 16bytes, i.e blocks required for 100 records
1600/512 = 3.12 = 4 blocks

So we have total 4+1 search instead of 25 blocks,

We can have another index for the indexed table which will have only four entries for each block we have in index at level 1. 
This index will only have one entry per block. 
The Level 2 index acts as a table of contents for those 4 blocks. It only needs 4 entries (one for each block), tracking the maximum ID in that block
:Max ID  Points to Level 1 Block
  25      Block A
  50      Block B
  75      Block C
  100     Block D
If say we need to read row 60. We go to 25 no, 50 no, 75 yes as it has entries between 50 and 75.
Then we go to Level 1 Block C, and here we need to check entire block for location of the actual row

The physical location is stored using a specialized data type usually called a RID (Record ID) or TID (Tuple ID).
What is a RID / TID? (8 Bytes)
A RID is not a single number like an integer. It is a composite data type (a struct or byte array) that contains two precise coordinates:
Page/Block Number:
The unique ID of the physical disk block where the row lives.
Slot Number (Offset): The specific row number inside that disk block.
4 Bytes for the Block Number (can address billions of blocks).
2 Bytes for the Slot/Offset Number (can address thousands of rows per block).
2 Bytes for internal flags or table/file identifiers.

DISK SWAPPING ?


Now M Way Seach Tree, but why? because it is related to indexing

M Way Search Tree - just think of it like BST but more keys per node. 
M no of children each node can have
M-1 Keys

Node for BST : |left| key | right|
Node for 4-way  BST:
|childNode1| key1 | childNode2| key2 | childNode3| key3 | childNode4| 

Now to use this for indexing
|childNode1| key1 |RecordPointer1| childNode2| key2 | RecordPointer2 | childNode3| RecordPointer3 | key3 | childNode4| RecordPointer4|




B Trees : M-way Search Trees with Rules
If the value of m = 10 then m/2 (ceil) children must be there for every node
Root doest follow this, i.e root can have min 2 children, i.e one key (because 2 children nodes will have val less than and greater than the key)
All Leaf Nodes at same level
BOTTOM UP CREATION PROCESS


Support 











