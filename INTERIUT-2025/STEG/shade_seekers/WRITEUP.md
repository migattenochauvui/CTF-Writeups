# Shade seekers

Lorsqu'on regarde avec pngcheck (ou qu'on upload le fichier sur Aperisolve), on se rend compte qu'il y a des valeurs étranges dans la palette de couleurs:
```
$ pngcheck -7cpqtv image.png 

File: image.png (755531 bytes)
  chunk IHDR at offset 0x0000c, length 13
    1024 x 1024 image, 8-bit palette, non-interlaced
  chunk PLTE at offset 0x00025, length 72: 24 palette entries
     0:  ( 97,  0,  0) = (0x61,0x00,0x00)
     1:  (  0, 87,  0) = (0x00,0x57,0x00)
     2:  (  0,  0, 53) = (0x00,0x00,0x35)
     3:  ( 48,  0,  0) = (0x30,0x00,0x00)
     4:  (  0, 90,  0) = (0x00,0x5a,0x00)
     5:  (  0,  0, 88) = (0x00,0x00,0x58)
     6:  ( 74,  0,  0) = (0x4a,0x00,0x00)
     7:  (  0,112,  0) = (0x00,0x70,0x00)
     8:  (  0,  0,100) = (0x00,0x00,0x64)
     9:  ( 88,  0,  0) = (0x58,0x00,0x00)
    10:  (  0, 82,  0) = (0x00,0x52,0x00)
    11:  (  0,  0, 55) = (0x00,0x00,0x37)
    12:  ( 99,  0,  0) = (0x63,0x00,0x00)
    13:  (  0, 51,  0) = (0x00,0x33,0x00)
    14:  (  0,  0, 82) = (0x00,0x00,0x52)
    15:  (108,  0,  0) = (0x6c,0x00,0x00)
    16:  (  0, 90,  0) = (0x00,0x5a,0x00)
    17:  (  0,  0,122) = (0x00,0x00,0x7a)
    18:  ( 74,  0,  0) = (0x4a,0x00,0x00)
    19:  (  0, 72,  0) = (0x00,0x48,0x00)
    20:  (  0,  0, 77) = (0x00,0x00,0x4d)
    21:  (110,  0,  0) = (0x6e,0x00,0x00)
    22:  (  0, 48,  0) = (0x00,0x30,0x00)
    23:  (  0,  0, 61) = (0x00,0x00,0x3d)
  chunk IDAT at offset 0x00079, length 755390
    zlib: deflated, 32K window, maximum compression
  chunk IEND at offset 0xb8743, length 0
No errors detected in image.png (4 chunks, 27.9% compression).
```

On extrait les valeurs étranges des champs RGB et ce qui donne:
`97,87,53,48,90,88,74,112,100,88,82,55,99,51,82,108,90,122,74,72,77,110,48,61`

On peut se douter que c'est de l'ascii ce qui donne une fois converti donne:
`aW50ZXJpdXR7c3RlZzJHMn0=`

On remarque que c'est du base64:
```
 echo aW50ZXJpdXR7c3RlZzJHMn0= | base64 --decode 
```

Ce qui nous donne le flag:
```
interiut{steg2G2}
```
