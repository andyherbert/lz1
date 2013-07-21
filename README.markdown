[This article original appeared on andyh.org.](http://andyh.org/LZ1.html)

Abraham Lempel and Jacob Ziv are considered to be pioneers in the field of data compression techniques, this is due to two academic papers they jointly published in the late 1970s that outlined a technique to efficiently compress data without any loss of information. For this article I'll be looking at the algorithm outlined in the first paper, which is commonly known known as LZ1.

LZ1 looks for repeated sequences of data, and records references to these duplicated chunks by storing pointers alongside the original information. These pointers are then used to rebuild the data back to its original structure. It works as follows: bytes are read in a sequence from first to last, the location of the byte currently being inspected is known as the 'coding point'. Each byte at the coding point is compared to information preceding it, in a buffer known as a 'window', this window would not be expected to match the whole size of the data, but it may extend to more than a few thousand bytes. If one, or many bytes within this window match the sequence currently located at the coding point, then a pointer is saved to the compressed data. This pointer consists of an offset from the current coding point to the repeated data, and how many bytes have been matched. This value is subsequently described as the pointer's 'length'. When the pointer is recorded, the byte immediately after the matched sequence at the coding point is also stored. If no match was found within the window, which would always happen when the first element is read from a file, then a null pointer is saved along with the byte at the coding point is saved with it. So whether we find a match in our window or not, we always save a pointer alongside a single byte as we work our way through the data.

So take the following example, a string of 'A's, followed by a 'B', and finally a 'C':

    AAAAAAAABC

The first 'A' is inspected, and the window contains no data at this point, so a null pointer is recorded alongside the byte at the coding point, i.e. the letter A. The coding point then moves to the next byte, the second 'A'. The algorithm has seen this before, namely in the previous byte, then it looks at how many bytes that follows also match if we were to advance the coding point whilst still referring back to the offset, which at the moment has the value of one. This is where the algorithm can be quite clever and match beyond the current coding point; currently the pointer is located at the first byte, and the coding point at the second, both are incremented whilst both the bytes located at each match. In this case, the total amount of bytes that are equivalent is seven, this will be until the 'B' is reached. If at this point we had already read more data, and therefore the window would be larger, then we would perform the same matching operation further down the data for the entire size of the window. Since this is our only, and therefore longest match, a pointer with the offset of one and a length of seven is written to the compressed data, as is next byte at the coding point beyond the pointer length, the letter B. And now the coding point moves past the 'B', and is inspected the final byte. It hasn't seen a 'C' before in the window, so a null-pointer is saved along with the letter C. It's important to avoid pushing the coding point past the length of the data when storing the final pointer and byte. To illustrate this scenario, consider this example: if our piece of data was composed only of eight 'A's, without the 'B', or 'C', then the initial 'A' would be recorded with a null-pointer, as previously noted, then a pointer to the first 'A' with a pointer length of six, with the final 'A' alongside it. The reason we would give the pointer a length of six and not seven is because we would still need to record the final byte along with the pointer, and matching seven characters in our pointer would force the coding point to go beyond the length of the data because an additional byte is expected with the final pointer.

Decompressing is an extremely simple task, the coding point and window are still held in memory whilst the data is being decoded. When a pointer offset is encountered, the data at the pointer offset is copied to the current coding point for however many times have been recorded by the pointer length, after this, the byte held with the pointer is then inserted at the coding point. So for our first example, the null-pointer is found and ignored, and then the first byte, the 'A', is read and inserted at the coding point. The second pointer is read, and we can see that the offset refers back to the previous character, so we copy everything we've already inserted at our previous coding point for the next seven iterations. This will give us seven 'A's. Alongside the pointer, the next character is given, 'B', so we add that at our coding point, and advance the coding point again. Finally, a null-pointer is discovered along with the final byte, a 'C', so we ignore the pointer and add the byte at the coding point. After this, our original data structure has successfully been rebuilt.

Adding pointers to the compressed data incurs a cost as well as a benefit. If the pointer is too large when stored in the compressed data, then it would become less useful. For this reason the size of the window and the length of bytes are also limited. For the code examples that follow, I've chosen to store the pointers a 16-bit byte, twelve bits specify a pointer offset, and four for the length. This results in a window of 4096 bytes, and a maximum pointer length of fifteen bytes. After experimenting with various combinations of size and length, a 16-bit pointer with these dimensions appears to be a beneficial trade-off. For most text files it's unlikely that a sequence longer than fifteen bytes would be repeated - although this would be more suitable for machine readable files with long portions of repeated data. A window of 4096 bytes also gives a good chance to find repeated data, anything smaller than this appears to negatively impact the size of the compressed data. Having a window that is excessively large also increases the time it takes to compress the data, as ultimately more data is compared. There are also memory constraints to consider too if the compression and decompression routines only store the window when performing its task.

In the code below, the compression routine accepts the following arguments: a pointer to a sequence of bytes for compression, the size of the uncompressed data, and lastly, a pointer to the location in memory to store the output. For decompressing, only a pointer to the compressed data, and a pointer to where the uncompressed data will be held are needed. The reason why the uncompressed data size is not required in the latter routine is because when the compressed data is initially written, the final uncompressed length is recorded at the start as a 32-bit unsigned integer, so the decompression routine knows when to stop decoding data by comparing the final size of the data to the current value of the coding point.

    #include <stdint.h>
    
    uint32_t lz77_compress (uint8_t *uncompressed_text, uint32_t uncompressed_size, uint8_t *compressed_text)
    {
        uint8_t pointer_length, temp_pointer_length;
        uint16_t pointer_pos, temp_pointer_pos, output_pointer;
        uint32_t compressed_pointer, output_size, coding_pos, output_lookahead_ref, look_behind, look_ahead;
        
        *((uint32_t *) compressed_text) = uncompressed_size;
        compressed_pointer = output_size = 4;
        
        for(coding_pos = 0; coding_pos < uncompressed_size; ++coding_pos)
        {
            pointer_pos = 0;
            pointer_length = 0;
            for(temp_pointer_pos = 1; (temp_pointer_pos < 4096) && (temp_pointer_pos <= coding_pos); ++temp_pointer_pos)
            {
                look_behind = coding_pos - temp_pointer_pos;
                look_ahead = coding_pos;
                for(temp_pointer_length = 0; uncompressed_text[look_ahead++] == uncompressed_text[look_behind++]; ++temp_pointer_length)
                    if(temp_pointer_length == 15)
                        break;
                if(temp_pointer_length > pointer_length)
                {
                    pointer_pos = temp_pointer_pos;
                    pointer_length = temp_pointer_length;
                    if(pointer_length == 15)
                        break;
                }
            }
            coding_pos += pointer_length;
            if(pointer_length && (coding_pos == uncompressed_size))
            {
                output_pointer = (pointer_pos << 4) | (pointer_length - 1);
                output_lookahead_ref = coding_pos - 1;
            }
            else
            {
                output_pointer = (pointer_pos << 4) | pointer_length;
                output_lookahead_ref = coding_pos;
            }
            *((uint32_t *) (compressed_text + compressed_pointer)) = output_pointer;
            compressed_pointer += 2;
            *(compressed_text + compressed_pointer++) = *(uncompressed_text + output_lookahead_ref);
            output_size += 3;
        }
        
        return output_size;
    }
    
    uint32_t lz77_decompress (uint8_t *compressed_text, uint8_t *uncompressed_text)
    {
        uint8_t pointer_length;
        uint16_t input_pointer, pointer_pos;
        uint32_t compressed_pointer, coding_pos, pointer_offset, uncompressed_size;
        
        uncompressed_size = *((uint32_t *) compressed_text);
        compressed_pointer = 4;
        
        for(coding_pos = 0; coding_pos < uncompressed_size; ++coding_pos)
        {
            input_pointer = *((uint32_t *) (compressed_text + compressed_pointer));
            compressed_pointer += 2;
            pointer_pos = input_pointer >> 4;
            pointer_length = input_pointer & 15;
            if(pointer_pos)
                for(pointer_offset = coding_pos - pointer_pos; pointer_length > 0; --pointer_length)
                    uncompressed_text[coding_pos++] = uncompressed_text[pointer_offset++];
            *(uncompressed_text + coding_pos) = *(compressed_text + compressed_pointer++);
        }
        
        return coding_pos;
    }
    

I have also included the following routines to load and save files locally, as well as an enttry-point function to demonstrate the program compressing its own source file, then output it to a file, and then finally decompress it as a copy of the original. For this to work correctly all the quoted code must be held in a file named 'lz.c', although hopefully this will be clear from reading the code. When passing memory to the compression routine, this code only allocates 640KB, which should be enough for anyone. Obviously this should be increased when dealing with larger files.

For reference, the original size of the uncompressed code is 5111 bytes, compressed this becomes 1834 bytes - a total saving of 3277, or 64% of the original.

    #include <stdlib.h>
    #include <stdio.h>
    #include <stdint.h>
    
    long fsize (FILE *in)
    {
        long pos, length;
        pos = ftell(in);
        fseek(in, 0L, SEEK_END);
        length = ftell(in);
        fseek(in, pos, SEEK_SET);
        return length;
    }
    
    uint32_t file_lz77_compress (char *filename_in, char *filename_out)
    {
        FILE *in, *out;
        uint8_t *uncompressed_text, *compressed_text;
        uint32_t uncompressed_size, compressed_size;
        
        in = fopen(filename_in, "r");
        if(in == NULL)
            return 0;
        uncompressed_size = fsize(in);
        uncompressed_text = malloc(uncompressed_size);
        if((uncompressed_size != fread(uncompressed_text, 1, uncompressed_size, in)))
            return 0;
        fclose(in);
        
        compressed_text = malloc(655360);
        
        compressed_size = lz77_compress(uncompressed_text, uncompressed_size, compressed_text);
        
        out = fopen(filename_out, "w");
        if(out == NULL)
            return 0;
        if((compressed_size != fwrite(compressed_text, 1, compressed_size, out)))
            return 0;
        fclose(out);
        
        return compressed_size;
    }
    
    uint32_t file_lz77_decompress (char *filename_in, char *filename_out)
    {
        FILE *in, *out;
        uint32_t compressed_size, uncompressed_size;
        uint8_t *compressed_text, *uncompressed_text;
        
        in = fopen(filename_in, "r");
        if(in == NULL)
            return 0;
        compressed_size = fsize(in);
        compressed_text = malloc(compressed_size);
        if(fread(compressed_text, 1, compressed_size, in) != compressed_size)
            return 0;
        fclose(in);
        
        uncompressed_size = *((uint32_t *) compressed_text);
        uncompressed_text = malloc(uncompressed_size);
        
        if(lz77_decompress(compressed_text, uncompressed_text) != uncompressed_size)
            return 0;
            
        out = fopen(filename_out, "w");
        if(out == NULL)
            return 0;
        if(fwrite(uncompressed_text, 1, uncompressed_size, out) != uncompressed_size)
            return 0;
        fclose(out);
        
        return uncompressed_size;
    }
        
    int main (int argc, char const *argv[])
    {
        FILE *in;
        in = fopen("./lz.c", "r");
        if(in == NULL)
            return 0;
        printf("Original size: %ld\n", fsize(in));
        fclose(in);
        printf("Compressed: %u\n", file_lz77_compress("./lz.c", "lz.c.z77"));
        printf("Decompressed: %u", file_lz77_decompress("./lz.c.z77", "lz-2.c"));
        return 0;
    }
    



    
    the_odyssey.txt:  678171
    
    Compressed (1):   680083
    Compressed (2):   437965
    Compressed (3):   372199 * 54.88%
    Compressed (4):   399466
    Compressed (5):   444541
    Compressed (6):   497605
    Compressed (7):   561688
    Compressed (8):   639040
    Compressed (9):   734527
    Compressed (10):  854881
    Compressed (11): 1004944
    Compressed (12): 1200883
    Compressed (13): 1457752
    Compressed (14): 1831507
    Compressed (15): 1993543
    
    lz.c:             5872
    
    Compressed (1):   6112
    Compressed (2):   3994
    Compressed (3):   2707
    Compressed (4):   2044
    Compressed (5):   1894 * 32.25%
    Compressed (6):   2032
    Compressed (7):   2485
    Compressed (8):   3124
    Compressed (9):   3931
    Compressed (10):  5362
    Compressed (11):  7366
    Compressed (12): 10879
    Compressed (13): 12904
    Compressed (14): 14251
    Compressed (15): 14998
    
    mach_kernel:      8194272
    
    Compressed (1):   8566924
    Compressed (2):   6171100
    Compressed (3):   5012707
    Compressed (4):   4768687 * 58.20%
    Compressed (5):   4912633
    Compressed (6):   5261968
    Compressed (7):   5804539
    Compressed (8):   6590989
    Compressed (9):   7682254
    Compressed (10):  9311959
    Compressed (11): 11194129
    Compressed (12): 13686307
    Compressed (13): 16359478
    Compressed (14): 18106570
    Compressed (15): 18874819

---

[This article original appeared on andyh.org.](http://andyh.org/LZ1-Postscript.html)

After toying with the LZ1 algorithm, I began to wonder what the benefits would be of using different pointer lengths on various types of files. I decided that the first file to test would be a large plain-text document, so I chose The Odyssey by Homer, the second would be a much smaller file, the source code of the program itself, and finally an arbitrary binary file, mach_kernel, an operating system kernel found in the root directory of OS X. The results that follow include the original file-size in bytes, and quotes the bit-width of the pointer length value used for each test in brackets. The results are also measured in bytes, and the best result is marked with an asterisk. The value after the asterisk denotes the size of the compressed file as a percentage of the original.

    The Odyssey:      678171             lz.c:             5872            mach_kernel:      8194272
                                                                           
    Compressed (1):   680083             Compressed (1):   6112            Compressed (1):   8566924
    Compressed (2):   437965             Compressed (2):   3994            Compressed (2):   6171100
    Compressed (3):   372199 * 54.88%    Compressed (3):   2707            Compressed (3):   5012707
    Compressed (4):   399466             Compressed (4):   2044            Compressed (4):   4768687 * 58.20%
    Compressed (5):   444541             Compressed (5):   1894 * 32.25    Compressed (5):   4912633
    Compressed (6):   497605             Compressed (6):   2032            Compressed (6):   5261968
    Compressed (7):   561688             Compressed (7):   2485            Compressed (7):   5804539
    Compressed (8):   639040             Compressed (8):   3124            Compressed (8):   6590989
    Compressed (9):   734527             Compressed (9):   3931            Compressed (9):   7682254
    Compressed (10):  854881             Compressed (10):  5362            Compressed (10):  9311959
    Compressed (11): 1004944             Compressed (11):  7366            Compressed (11): 11194129
    Compressed (12): 1200883             Compressed (12): 10879            Compressed (12): 13686307
    Compressed (13): 1457752             Compressed (13): 12904            Compressed (13): 16359478
    Compressed (14): 1831507             Compressed (14): 14251            Compressed (14): 18106570
    Compressed (15): 1993543             Compressed (15): 14998            Compressed (15): 18874819
    

The results are not really that surprising, the most gains to be found are with a bit-width of between three to five. The source code file, which contains many repeated references to function calls and variable names, produced the best results from the three files.

I've included the code for the updated compress and decompress functions, which now allow an extra parameter to specify the pointer length bit-width. The pointer size still remains as a 16-bit integer, as anything smaller would negatively impact the efficiency of the program. Additionally, when the file is compressed, the pointer length is written as an 8-bit integer immediately after the uncompressed file-length value, and before the raw compressed data. This way, the bit-width of the pointer length can be stored dynamically for different files. Another small change was made to both routines - a pointer length of zero would never be used, so zero now represents one, one for two, and so on. Both of these alterations breaks compatibility with files written by the old routines.

    #include <stdint.h>
    #include <math.h>
    
    uint32_t lz77_compress (uint8_t *uncompressed_text, uint32_t uncompressed_size, uint8_t *compressed_text, uint8_t pointer_length_width)
    {
        uint16_t pointer_pos, temp_pointer_pos, output_pointer, pointer_length, temp_pointer_length;
        uint32_t compressed_pointer, output_size, coding_pos, output_lookahead_ref, look_behind, look_ahead;
        uint16_t pointer_pos_max, pointer_length_max;
        pointer_pos_max = pow(2, 16 - pointer_length_width);
        pointer_length_max = pow(2, pointer_length_width);
        
        *((uint32_t *) compressed_text) = uncompressed_size;
        *(compressed_text + 4) = pointer_length_width;
        compressed_pointer = output_size = 5;
    
        for(coding_pos = 0; coding_pos < uncompressed_size; ++coding_pos)
        {
            pointer_pos = 0;
            pointer_length = 0;
            for(temp_pointer_pos = 1; (temp_pointer_pos < pointer_pos_max) && (temp_pointer_pos <= coding_pos); ++temp_pointer_pos)
            {
                look_behind = coding_pos - temp_pointer_pos;
                look_ahead = coding_pos;
                for(temp_pointer_length = 0; uncompressed_text[look_ahead++] == uncompressed_text[look_behind++]; ++temp_pointer_length)
                    if(temp_pointer_length == pointer_length_max)
                        break;
                if(temp_pointer_length > pointer_length)
                {
                    pointer_pos = temp_pointer_pos;
                    pointer_length = temp_pointer_length;
                    if(pointer_length == pointer_length_max)
                        break;
                }
            }
            coding_pos += pointer_length;
            if((coding_pos == uncompressed_size) && pointer_length)
            {
                output_pointer = (pointer_length == 1) ? 0 : ((pointer_pos << pointer_length_width) | (pointer_length - 2));
                output_lookahead_ref = coding_pos - 1;
            }
            else
            {
                output_pointer = (pointer_pos << pointer_length_width) | (pointer_length ? (pointer_length - 1) : 0);
                output_lookahead_ref = coding_pos;
            }
            *((uint16_t *) (compressed_text + compressed_pointer)) = output_pointer;
            compressed_pointer += 2;
            *(compressed_text + compressed_pointer++) = *(uncompressed_text + output_lookahead_ref);
            output_size += 3;
        }
        
        return output_size;
    }
    
    uint32_t lz77_decompress (uint8_t *compressed_text, uint8_t *uncompressed_text)
    {
        uint8_t pointer_length_width;
        uint16_t input_pointer, pointer_length, pointer_pos, pointer_length_mask;
        uint32_t compressed_pointer, coding_pos, pointer_offset, uncompressed_size;
        
        uncompressed_size = *((uint32_t *) compressed_text);
        pointer_length_width = *(compressed_text + 4);
        compressed_pointer = 5;
        
        pointer_length_mask = pow(2, pointer_length_width) - 1;
        
        for(coding_pos = 0; coding_pos < uncompressed_size; ++coding_pos)
        {
            input_pointer = *((uint16_t *) (compressed_text + compressed_pointer));
            compressed_pointer += 2;
            pointer_pos = input_pointer >> pointer_length_width;
            pointer_length = pointer_pos ? ((input_pointer & pointer_length_mask) + 1) : 0;
            if(pointer_pos)
                for(pointer_offset = coding_pos - pointer_pos; pointer_length > 0; --pointer_length)
                    uncompressed_text[coding_pos++] = uncompressed_text[pointer_offset++];
            *(uncompressed_text + coding_pos) = *(compressed_text + compressed_pointer++);
        }
        
        return coding_pos;
    }
