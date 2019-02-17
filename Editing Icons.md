# Editing Icons

## Software
  - LibreSprite / Aseprite
  - ImageMagick

*LibreSprite* can be used to create and edit icons for Add Ins.  
Icon must be a monochrome bitmap file (.bmp) of size `30x19`.

## Note for Casio SDK Users
When editing the reference/sample icon provided during the creation
of a new project, *LibreSprite* will be able to open it and edit it but the
newly saved icon won't be able to be used straight away by the SDK.  
To be able to use it you must pass it through ImageMagick first.  
Execute:  
`convert <edited image>.bmp -monochrome <output image>.bmp`  
Be aware that after this step, this "converted" file won't be able to be
opened by *LibreSprite* so save your design separately.  
Be sure to either name it `MainIcon.bmp` or change the file in Project Settings
inside the SDK.
