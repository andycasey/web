---
title: Reading FITS Headers Fast
description: "How to extract FITS header values without parsing the whole file."
date: 2026-02-07
draft: true
authors:
  - name: Andy Casey
    link: https://astrowizici.st
    image: ../../me.png
tags:
  - astronomy
  - python
  - performance
---

When processing thousands (or millions) of FITS files, reading headers with astropy becomes a bottleneck—especially for compressed files. Each call to `fits.getheader()` on a gzipped FITS file must decompress the entire file, even if you only need a few keywords from the header.

For [Almanac](https://github.com/andycasey/almanac), which catalogs millions of SDSS APOGEE exposures, I needed something faster.

## The trick: decompress only what you need

FITS headers are ASCII text stored at the beginning of the file in 80-character fixed-width records. You don't need to parse the full FITS structure to read them—you just need the first few kilobytes.

```python
from subprocess import check_output

def get_fits_headers(path, keywords, head_bytes=20_000):
    """
    Extract specific FITS header values from the primary HDU.
    
    For compressed files, this is ~20x faster than astropy.io.fits
    because we only decompress the first few KB.
    
    Parameters
    ----------
    path : str
        Path to FITS file (supports .fits, .fits.gz, .fits.fz)
    keywords : tuple
        Header keywords to extract (e.g., ('EXPTIME', 'OBJECT'))
    head_bytes : int
        Bytes to read from start of file (20KB covers most headers)
    
    Returns
    -------
    dict
        Mapping of keyword -> value (None if not found)
    """
    keys_pattern = "|".join(keywords)
    
    if path.endswith('.gz'):
        # Decompress only first N bytes
        cmd = f"gzip -dc '{path}' | head -c {head_bytes} | " \
              f"hexdump -e '80/1 \"%_p\" \"\\n\"' | egrep '{keys_pattern}'"
    elif path.endswith('.fz'):
        # fpack compressed
        cmd = f"funpack -S '{path}' | head -c {head_bytes} | " \
              f"hexdump -e '80/1 \"%_p\" \"\\n\"' | egrep '{keys_pattern}'"
    else:
        # Uncompressed - read directly
        cmd = f"hexdump -n {head_bytes} -e '80/1 \"%_p\" \"\\n\"' '{path}' | " \
              f"egrep '{keys_pattern}'"
    
    try:
        output = check_output(cmd, shell=True, text=True)
        return _parse_fits_header_lines(output.strip().split("\n"), keywords)
    except:
        return {k: None for k in keywords}


def _parse_fits_header_lines(lines, keywords):
    """Parse FITS header lines in KEY = VALUE format."""
    values = {k: None for k in keywords}
    
    for line in lines:
        if '=' not in line:
            continue
        
        key = line[:8].strip()
        if key not in keywords:
            continue
            
        # Extract value, handling comments after /
        value_part = line[10:]  # Skip 'KEY     = '
        if '/' in value_part:
            value_part = value_part.split('/')[0]
        
        value = value_part.strip().strip("'")
        values[key] = value
    
    return values
```

### How it works

1. **`gzip -dc | head -c 20000`** decompresses only the first 20KB
2. **`hexdump -e '80/1 "%_p" "\n"'`** formats as 80-char lines (FITS record size)
3. **`egrep`** filters to only lines containing your keywords
4. We parse the `KEY = VALUE` format, stripping comments

## Benchmarks

Tested on a 60 MB gzipped FITS file (4096×4096 float32 image):

| Method | Time per file | Speedup |
|--------|--------------|---------|
| `fits.getheader()` (compressed) | 115 ms | 1× |
| `gzip -dc \| head \| hexdump` | 5.3 ms | **22×** |

For uncompressed files, astropy is actually faster (~0.3 ms) because it efficiently reads just the header block. The speedup comes from avoiding full decompression of compressed files.

**When to use this:**
- Processing many compressed FITS files
- You know which keywords you need upfront
- Keywords are in the primary header

## Reading extension headers

What if you need headers from the second HDU? You'll need to calculate the offset and skip to it.

FITS files are structured as: `[Primary Header][Primary Data][Ext1 Header][Ext1 Data]...`

Each section is padded to 2880-byte boundaries. To find extension headers, we calculate the primary HDU size from `BITPIX` and `NAXISn` keywords:

```python
def get_extension_header(path, ext=1, max_read=500_000):
    """
    Read header from extension HDU by calculating offset.
    
    Parameters
    ----------
    path : str
        Path to FITS file
    ext : int
        Extension number (1 = first extension after primary)
    max_read : int
        Maximum bytes to read (must cover primary + extension headers)
    
    Returns
    -------
    dict
        Header keyword-value pairs from the extension
    """
    is_compressed = path.endswith('.gz') or path.endswith('.fz')
    
    # Read enough bytes to cover headers
    if is_compressed:
        from subprocess import check_output
        raw = check_output(f"gzip -dc '{path}' | head -c {max_read}", shell=True)
    else:
        with open(path, 'rb') as f:
            raw = f.read(max_read)
    
    # Calculate primary HDU size
    offset = _calculate_hdu_size(raw)
    
    # Skip to extension (repeat for ext > 1)
    for _ in range(ext - 1):
        offset += _calculate_hdu_size(raw[offset:])
    
    # Parse extension header
    return _parse_header_bytes(raw[offset:])


def _calculate_hdu_size(raw_bytes):
    """Calculate total HDU size from header bytes."""
    # Parse header to find BITPIX, NAXIS, NAXISn
    naxis = 0
    bitpix = 8
    axis_sizes = []
    header_end = 0
    
    for i in range(0, min(len(raw_bytes), 100_000), 80):
        line = raw_bytes[i:i+80].decode('ascii', errors='replace')
        
        if line.startswith('END'):
            header_end = i + 80
            break
        elif line.startswith('BITPIX'):
            bitpix = int(line.split('=')[1].split('/')[0])
        elif line.startswith('NAXIS   '):
            naxis = int(line.split('=')[1].split('/')[0])
        elif line.startswith('NAXIS') and '=' in line:
            axis_sizes.append(int(line.split('=')[1].split('/')[0]))
    
    # Header size: round up to 2880 boundary
    header_size = (header_end + 2879) // 2880 * 2880
    
    # Data size: |BITPIX|/8 × NAXIS1 × NAXIS2 × ...
    if naxis == 0:
        data_size = 0
    else:
        data_size = abs(bitpix) // 8
        for size in axis_sizes[:naxis]:
            data_size *= size
    
    # Data also padded to 2880 boundary
    padded_data = (data_size + 2879) // 2880 * 2880
    
    return header_size + padded_data


def _parse_header_bytes(raw_bytes, max_lines=500):
    """Parse header keywords from raw bytes."""
    results = {}
    for i in range(0, min(len(raw_bytes), max_lines * 80), 80):
        line = raw_bytes[i:i+80].decode('ascii', errors='replace')
        
        if line.startswith('END'):
            break
        if '=' in line:
            key = line[:8].strip()
            value = line[10:].split('/')[0].strip().strip("'")
            results[key] = value
    
    return results
```

### Caveats for extensions

- You need enough bytes to cover all HDUs up to your target (`max_read` parameter)
- For compressed files, this still requires decompressing up to that point
- Variable-length tables (heap) require additional offset calculations

## When astropy is better

This approach trades correctness for speed. Use astropy when you need:

- Extension headers from large files (seek is more efficient than decompress)
- Keywords with CONTINUE cards
- HIERARCH keywords
- Header history/comments
- Proper type conversion (int, float, bool)

## The takeaway

Sometimes the fastest way to process astronomical data is to remember that FITS is just a file format. When you need raw speed over correctness for every edge case, decompress only what you need and parse the ASCII directly.

For batch processing compressed files—common in survey astronomy—this can turn hours into minutes.
