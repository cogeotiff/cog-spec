## Cloud optimized GeoTIFF

### Definition

A cloud optimized GeoTIFF is a regular GeoTIFF file, aimed at being hosted on a HTTP file server, 
whose internal organization is friendly for consumption by clients issuing [HTTP GET range request](https://tools.ietf.org/html/rfc7233) 
("bytes: start_offset-end_offset" HTTP header).

It contains at its beginning the metadata of the full resolution imagery, followed by the optional presence 
of overview metadata, and finally the imagery itself. To make it friendly with streaming and progressive 
rendering, we recommand starting with the imagery of the smallest overview and finishing with the imagery 
of the full resolution level.

More formally, the structure of such a file is:

* TIFF / BigTIFF signature
* IFD ([Image File Directory](https://www.awaresystems.be/imaging/tiff/faq.html#q3) of full resolution image
* Values of TIFF tags that don't fit inline in the IFD directory, such as TileOffsets, TileByteCounts and GeoTIFF keys
* Optional: IFD (Image File Directory) of first overview (typically subsampled by a factor of 2), followed by the values of its tags that don't fit inline
* Optional: IFD (Image File Directory) of second overview (typically subsampled by a factor of 4), followed by the values of its tags that don't fit inline
* ...
* Optional: IFD (Image File Directory) of last overview (typically subsampled by a factor of 2N), followed by the values of its tags that don't fit inline
* Optional: tile content of last overview level
* ...
* Optional: tile content of first overview level
* Tile content of full resolution image.

### Unspecified points

* size of tile: 256 or 512 pixels are typical however
* compression methods allowed. typically, no compression, DEFLATE or LZW can be used for lossless, or JPEG for lossy. (note that DEFLATE while more efficient than LZW can cause compatibility issues with some software packages)

### How to generate it with GDAL

Given an input dataset in.tif with already generated internal or external overviews, a cloud optimized GeoTIFF can be generated with:

`gdal_translate in.tif out.tif -co TILED=YES -co COPY_SRC_OVERVIEWS=YES -co COMPRESS=LZW`

This will result in a images with tiles of dimension 256x256 pixel for main resolution, and 128x128 tiles for overviews.

For an image of 4096x4096 with 4 overview levels, the 5 IFDs and their TileOffsets and TileByteCounts tag data fit into the first 6KB of the file.

Note: for JPEG compression, the above method produce cloud optimized files only if using GDAL 2.2 
(or a dev version >= r36879). For older versions, the IFD of the overviews will be written towards 
the end of the file. A recent version of GDAL (2.2 or dev version >= r37257) built against internal 
libtiff (or libtiff >= 4.0.8) will also help reducing the amount of bytes read for JPEG compressed 
files with YCbCr subsampling.

### How to read it with GDAL

GDAL includes special filesystems that can read a file hosted on a HTTP/FTP server by chunks.

The base filesystem is [/vsicurl/](http://gdal.org/cpl__vsi_8h.html#a4f791960f2d86713d16e99e9c0c36258) (Virtual System Interface for Curl) and the filename it accepts are of the form "/vsicurl/http://example.com/path/to/some.tif". They can be used whereever GDAL expects a dataset / filename to be passed: gdalinfo, gdal_translate, GDALOpen() API, etc...

Currently /vsicurl/ uses 16 KB as the minimum unit for downloading with HTTP range requests, and a in-memory cache of up to 1000 16KB blocks, with a least recently used eviction strategy.

To minimize the total number of HTTP requests outside of the target GeoTIFF file, setting the GDAL_DISABLE_READDIR_ON_OPEN=YES and CPL_VSIL_CURL_ALLOWED_EXTENSIONS=tif environment variables/configuration options is recommended so as to avoid any side-car files (such a .ovr, .aux.xml, .aux, etc.) to be probed.

Running gdalinfo or GDALOpen() on such a cloud optimized GeoTIFF will retrieve all the metadata with a single HTTP request of 16 KB. When reading pixels in a tile, only the blocks of 16 KB intersecting the range of the tile will be downloaded.

For files hosted on Amazon S3 storage, with non-public sharing rights, [/vsis3/](http://www.gdal.org/cpl__vsi_8h.html#a5b4754999acd06444bfda172ff2aaa16) can be used.

### How to check if a GeoTIFF has a cloud optimization internal organization?
The [validate_cloud_optimized_geotiff.py](https://github.com/OSGeo/gdal/blob/master/gdal/swig/python/samples/validate_cloud_optimized_geotiff.py) script can be used to check that a (GeoTIFF) file follows the above described file structure

```
$ python validate_cloud_optimized_geotiff.py test.tif
```
or
```
$ python
import validate_cloud_optimized_geotiff.py
validate_cloud_optimized_geotiff.validate('test.tif')
```
