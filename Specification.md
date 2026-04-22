# 3D Tiles Archive Format v1.3

The 3D Tiles Archive format 1.3 is based on the ZIP file format as defined by the ISO/IEC 21320-1:2015 specification; see https://www.iso.org/standard/60101.html. It adds the Zstandard compression method defined in “APPNOTE - .Zip File Format Specification” version 6.3.9; see https://pkware.cachefly.net/webdocs/APPNOTE/APPNOTE-6.3.9.TXT. The Zstandard compression format itself is defined in IETF RFC 8474; see https://tools.ietf.org/html/rfc8478.

A 3D Tiles archive MUST be a valid ZIP file according to the ISO/IEC 21320-1:2015 specification, with the addition of the Zstandard compression method as defined in “APPNOTE - .Zip File Format Specification” version 6.3.9, noting that it MUST use compression method ID 93 to indicate that Zstandard was used.

> **Informative note**: _For optimal read performance, files in the archive read fastest when stored without compression. However, the Zstandard compression method provides a good trade-off between read performance and file size. For best compatibility with legacy software, the standard DEFLATE compression method provides the widest support, although it is the slowest of the three methods._

All Local File Headers in the ZIP file MUST have compressed file size, uncompressed file size, and CRC32 set in the header. Note: This restricts the maximum file size of files inside the archive to 4 GB.

The archive MUST contain a file named `tileset.json` at the root level, and that file MUST be a valid 3D Tiles tileset file; see https://github.com/CesiumGS/3d-tiles/tree/master/specification.

All references made in files inside the archive SHOULD be relative. This includes URIs inside 3D Tiles tilesets as well as any references made in any other documents in the archive. Note that relative references may be external to the current archive.

Note that using URI schemes such as http:// to refer to some external location may not be supported in all clients and may negatively impact the portability of the data.

All references SHOULD resolve within a given distribution unit (a known collection of 3tz container files) for maximum compatibility.

A 3tz archive MUST NOT be placed inside another 3tz archive.

Filenames and directory names inside a 3tz archive MUST NOT contain the substring `.3tz` or `.3dtiles.zip`.

The archive MUST use the `.3tz` file extension or the `.3dtiles.zip` file extension.

The archive SHOULD use the `application/vnd.maxar.archive.3tz+zip` media type.

The archive MUST contain a valid index file, and it MUST be stored uncompressed to improve load and read performance when the archive contains a very large number of files. The index file MUST be named `@3dtilesIndex1@` and MUST be the last file in the archive; that is, it MUST be the last entry in the ZIP Central Directory (see Zip File Format Specification).

The `@3dtilesIndex1@` file in the archive MUST NOT have an associated file comment in the ZIP Central Directory (see Zip File Format Specification).

## Reading and writing the index file

To generate the `@3dtilesIndex1@` index file, follow these steps.

1.	For each file in the archive (excluding the index file itself if present), find the offset to the corresponding Zip Local File Header (see Zip File Format Specification).
2.	Normalize each file path by:
    1.	Replacing backslashes with forward slashes
    2.	Dropping any leading forward slashes (e.g '/tileset.json' => 'tileset.json')
3.	Compute the MD5 128-bit hash for the normalized file path.
4.	Create an index entry with the hash (16 bytes) and the corresponding offset (8 bytes) for a total of 24 bytes.
5.	Insert the index entry into the index, sorted in ascending order by the 128-bit MD5 hash interpreted as two little-endian 64-bit unsigned integers, using the following pseudocode:

        function md5_less_than(md5 hashA, md5 hashB) {
          hiA = hashA.readUInt64LE(0)
          hiB = hashB.readUInt64LE(0)
          if hiA == hiB then
            lowA = hashA.readUInt64LE(8)
            lowB = hashB.readUInt64LE(8)
            return lowA < lowB
          else
            return hiA < hiB
        }

6.	Write the index to the `@3dtilesIndex1@` binary file in little-endian order, with no padding and no header.

To find a specific file in the archive using the `@3dtilesIndex1@` index:
1.	Normalize the given file path using the steps above.
2.	Compute the MD5 hash for the normalized file path.
3.	Search the index to find an entry with a matching MD5 hash. Since the index is sorted in ascending order by the MD5 hash (see above), implementations SHOULD use a binary search algorithm to quickly find the given index entry.
4.	If an index entry was found, read the Zip Local File Header (see Zip File Format Specification) at the entry offset.
5.	Verify that the found Local File Header is the correct one, since there may have been a hash collision.
a.	Compare the Local File Header filename with the normalized file path.
b.	If that’s a match, continue to step 6.
c.	Otherwise, find all index entries that have the same MD5 hash (they will be adjacent in the index table), then repeat steps 4 and 5 for them (excluding step 5c).
6.	Return the associated decompressed file contents (see Zip File Format Specification).

## Efficient reading of the 3D Tiles Archive Format

Readers of the 3D Tiles Archive Format SHOULD NOT use an off-the-shelf ZIP archive reader to open the archive. Instead, they SHOULD first scan a few hundred bytes from the end of the archive to find the last entry in the Central Directory. If the filename of the last entry matches the known 3D Tiles archive index filename defined by this specification, implementations SHOULD attempt to load the file contents using the information found in the Central Directory entry. They SHOULD then use the index to quickly find files inside the archive, as described above.

It is still possible to read the 3D Tiles Archive as a standard ZIP archive, noting that it might be much slower in comparison.

An alternative approach is to extract the index from the archive once using any standard ZIP software and then use it to read the archive.
