# LSB and DCT Steganography
This is an implementation of LSB and DCT steganography for COS 398.

## LSB Steganography
LSB steganography is a technique for hiding data in an image. The data is hidden in the least significant bit of each channel for each pixel. The least significant bit is the bit all the way to the right and changing it will have the smallest effect on byte values that are extracted from the image. This means that if you take the pixel shape of the image and multiply it by 3, for the RGB channels, you will get the number of bits that can be stored in the image.

![plot](startImages/Photo1.jpg)

The following image is one of the images used for the examples of how this code can be used. The size of the image is 744x1700, this means that we can store 1,264,800 bits in the image. Using UTF-8 encoding, we can store a maximum of 16,384 characters in the image. This is because each character is encoded in 8 bits, or one byte.

While 16,384 characters sounds like a large message, and for humans it is, the eventual goal is for this project is to be integrated into a larger project that can handle any type of binary file. This means that either the size of the image needs to change, or the encoding needs to change. So I changed the encoding. Instead of using just the least significant bit, I used the last two bit, doubling the number of bits that can be stored in the image. This means that the image now can store 2,529,600 bits or 32,768 bytes. 

### ___How it works___
The first step is to load both the image file and the message file. For the LSB portion of the project I used the basic PIL library to load the image becaue no heavy manipulation of the image was needed. The message file is loaded using the built in python library for reading files.

```python
Photo1 = Image.open("startImages/IMG_4279.JPG")

hiddenMessage = open('ThingsToHide/LoTR.txt', 'r').read()
```

The text information is loaded into python as a string, but we need the binary representation of the string. The built in python library method for converting strings to binary is called `bin`, but in python the `bin` function only works on integers. So we need to convert the string to an integer, and then convert that integer to binary. Strangly enough the first 0 was always missing, so I added it to make it decode back to the original string correctly.

```python
def string_to_binary(message):
    encodeM = str(message).encode()
    bin_int = int.from_bytes(encodeM, 'big')
    bin_string = bin(bin_int)[2:]
    return "0" + bin_string
```

Next we need to iterate over each pixel in the image and its 3 channels, converting each channel integer in to binary as we do making an easy to manipulate list. This list is then put into a method that will hide the message in the last two bits of each item in the list. As the list is iterated over, the last two bits are stripped off the channel integer and the first two bits of the message are added to the end. As this happens the message is decremented till there is nothing left.

```python
def encode_message(pixels, binString):
    # loop through each pixel
    for i in range(len(pixels)):
        for j in range(len(pixels[i])):
            if (binString == ''):
                return pixels
            # get the pixel value this is in binary already
            pixel = pixels[i][j]
            # get first two characters of binString to be implanted into pixel
            binStringC = binString[:2]
            # remove first two characters of binString to the message will be eaten while looping
            binString = binString[2:]
            # get the first 6 character of binC then add the two bits of binStringC to the end
            pixel = pixel[:6] + binStringC
            #set the new pixel value
            pixels[i][j] = pixel
    return pixels
```
The image is then saved to a new file. Below is an example of the first image with a part of the constitution encoded in it.

![plot](encodedImages/lsb_notEncyptedtest.png)

As you can see the image is visually indestiquishable from the original.

But I wanted to see how much information I could hid in a larder image, so I went and took some images with a lot of detail.

#### ___Original Image___
![plot](startImages/IMG_4279.JPG)

#### ___Encoded Image___
![plot](encodedImages/lsb_notEncypted.png)
The above image has the whole first Lord of the Rings book encoded in it, and it could store it about 7 times over.


## DCT Steganography
Discrete cosine transform is a bit more complicate the LSB encoding but it actually still uses a form of LSB. Instead of changing the the least significant bit in each of the channels of the pixels, you split the picture into 8x8 blocks and then perform a DCT on each block. The DCT is a mathematical operation that takes a matrix and transforms it into a new matrix. The DCT is a good way to hide information in an image because it is a simple operation that can be performed on any matrix. There are some downsides to the DCT method that will be talked about later, but it is still a good method to hide information.

The first thing that needs to be done, is make sure the image is a multiple of 8. If it is not, then we need to add padding to the image. Thankfully opencv has a function that can do this for us. After, like said above, we need to split the image into 8x8 blocks and then perform the DCT on each block and then run each block through a quantization matrix. The quantization matrix is a matrix that is used to map each transormation to a value between 0 and 255, the maximum value that can be stored in the RGB channels. Then you can preform the normal LSB encoding on each of those, and then do an inverse DCT on each of the blocks, and then reform the image out of the new blocks.
```python
def encode_image(self,img,message):
        
    #get length of message
    self.message = str(len(message))+'*'+message
        
    self.bitMess = self.toBits()
        
    #get size of image in pixels
    row,col = img.shape[:2]


    if((col/8)*(row/8)<len(message)):
        print("Error: Message too large to encode in image")
        return False
        
    #make divisible by 8x8
    if row%8 != 0 or col%8 != 0:
        img = cv2.resize(img,(col+(8-col%8),row+(8-row%8)))
        
    row,col = img.shape[:2]

    #split image into RGB channels
    bImg,gImg,rImg = cv2.split(img)
        
    #message to be hid in blue channel so converted to type float32 for dct function
    bImg = np.float32(bImg)
        
    #break into 8x8 blocks
    imgBlocks = [np.round(bImg[j:j+8, i:i+8]-128) for (j,i) in itertools.product(range(0,row,8),range(0,col,8))]
        
    #Blocks are run through DCT function
    dctBlocks = [np.round(cv2.dct(img_Block)) for img_Block in imgBlocks]
        
    #blocks then run through quantization table
    quantizedDCT = [np.round(dct_Block/quant) for dct_Block in dctBlocks]
        
    #set LSB in DC value corresponding bit of message
    messIndex = 0
    letterIndex = 0
        
    for quantizedBlock in quantizedDCT:
        #find LSB in DC coeff and replace with message bit
        DCoeff = quantizedBlock[0][0]
        DCoeff = np.uint8(DCoeff)
        DCoeff = np.unpackbits(DCoeff)
        DCoeff[7] = self.bitMess[messIndex][letterIndex]
        DCoeff = np.packbits(DCoeff)
        DCoeff = np.float32(DCoeff)
        DCoeff= DCoeff-255
        quantizedBlock[0][0] = DCoeff
        letterIndex = letterIndex+1
        if letterIndex == 8:
            letterIndex = 0
            messIndex = messIndex + 1
            if messIndex == len(self.message):
                break
    #blocks run inversely through quantization table
    sImgBlocks = [quantizedBlock *quant+128 for quantizedBlock in quantizedDCT]

    #puts the new image back together
    sImg=[]
        
    for chunkRowBlocks in self.chunks(sImgBlocks, col/8):
        for rowBlockNum in range(8):
            for block in chunkRowBlocks:
                sImg.extend(block[rowBlockNum])
        
    sImg = np.array(sImg).reshape(row, col)
        
    #converted from type float32
    sImg = np.uint8(sImg)
        
    #show(sImg)
    sImg = cv2.merge((sImg,gImg,rImg))
        
    return sImg
```
#### ___Original Image___
![plot](startImages/IMG_4279.JPG)
#### ___Encoded Image___
![plot](encodedImages/dct_Encoded.png)
Now obviously, the image is different, visually different. This is because the information is only hidden in one channel of the image. This can shift the color vaules of that channel, giving it a hue shift. However, you can use this to your advantage by deciding which channel you want to hide the information in, based on the picture.
#### ___Bad Channel___
![plot](encodedImages/dct_Encoded_bad_Color.png)
Above is the first image that that was shown after a blue channel dct implantation, the original image contains many reds and browns. This mans that if theres a blue shift it will be very noticeable. So by picking an image with that compliemnts your shift, you can reduce the noticabliilty of the change. On top of that in todays age with all the social media and filters that people use, you can select a channel that gives a desitred "filter" effect to the image, like in the image of the plane, giving you an other level of obscurity. If no one thinks the image looks out of place they won't look into it. There is however another downside to this method, it can not hide as much information as the LSB method, but it does make it harder to find so there are trade offs. 
## Encryption
I know that steganography is about security through obscurity, but I wanted to try and implement encryption before I placed it in the image. I tried a few methods, such as AES and Salsa20 but was unable to have any success with them. With both AES and Salsa20, the issue occured between the the encryption and encoding. I tried a few different python packages, but ultimatly when the message string was turned into bytes, padded to fit the encryption block and then encrypted, and then encoded, the message was not able to be decoded back into the encrypted text. I spent days going over the encryption algorithm and both the encoding algorithms and was not able to figure out where exactly between those two points the issue was occuring. One possiblity is that the python packages for encryption are really just wrappers for C++ code, and the issue could be happening with types and not reading the types correctly, which can be a big issue with python because it's non type language. To fix this I would either have to change languages or figure out some way to handle the raw bytes correctly. So instead while not very advanced, I was able to find an virtual enigma machine package for python that I could use to encrypt the message. While not modern day encryption, its still a form of encryption, and I thought it would be fun to try. The downside of this is that in order to use it you must normailze the string this means no puntuation, no spaces, no symbols, and no numbers. On top of that the message is split into groups of 5 characters because of how the enigma machine works. This means that when you decrypt the message it can be hard to read, as shown in the DCT_decodedMessage_With_Encryption.txt