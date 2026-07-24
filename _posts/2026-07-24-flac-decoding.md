# FLAC Decoding Notes

Writing this to record down the idea of flac decoding.

## header and streaminfo block

* fetch first four byte, match it with 'flac' in ascii
```
    const char *filename = "test_audio.flac";

    FILE *file_in = fopen(filename, "rb");
    if (file_in == NULL) {
        printf("file not found\n");
        return 0;
    }

    uint32_t flac_header;
    fread(&flac_header, sizeof(uint32_t), 1, file_in);

    // "flac" in little endian
    if (flac_header != 0x43614c66) { 
        printf("header incorrect, 0x%02x\n", flac_header);
        return 0;
    } else {
        printf("header ok\n");
    }
```

* fetch 38 byte of metadata

```
    uint8_t metadata[38];

    fread(&metadata, sizeof(uint8_t), 38, file_in);
```

* the most significant bit of metadata[0] must be 0
* we can read its type, length, min block size, max block size, sample rate, number of channel and bit depth of the audio

```
    uint8_t type = metadata[0] & 0x7F;
    printf("type: 0x%02x\n", type);

    uint32_t length = (metadata[1] << 16) | (metadata[2] << 8) | metadata[3];
    printf("length: 0x%02x\n", length);

    uint16_t min_block_size = (metadata[4] << 8) | metadata[5];
    uint16_t max_block_size = (metadata[6] << 8) | metadata[7];
    printf("min block size: %d, max block size: %d\n", min_block_size, max_block_size);

    // start from 14th block, 20 bit for sample rate
    uint32_t sample_rate = (metadata[14] << 12) | (metadata[15] << 4) | (metadata[16] >> 4);
    printf("sample rate: %d\n", sample_rate);

    uint8_t num_channel = ((metadata[16] & 0x0E) >> 1) + 1;
    printf("num channel: %d\n", num_channel);

    uint8_t bit_depth = (((metadata[16] & 0x01) << 4) | (metadata[17] >> 4)) + 1;
    printf("bit depth: %d\n", bit_depth);
```
* from the number of samples, we can know the length of the audio also

```
    int minute, second;
    minute = num_of_samples / (sample_rate * 60) ;
    second = (num_of_samples - (uint64_t)minute * sample_rate * 60) / sample_rate;
    printf("audio length: %d:%d\n", minute, second);
```

* here is the streaminfo data of a 24 bit 96khz flac audio

```
header ok
not last block, ok
type: 0x00
length: 0x22
min block size: 1920, max block size: 1920
sample rate: 96000
num channel: 2
bit depth: 24
num of samples: 20124784
audio length: 3:29
```

* in the flac encoding tools, there is default selection of block size

```
       -b #, --blocksize=#
              Specify the block size in samples.  The default is 1152  for  -l
              0,  else  4608;  must be one of 192, 576, 1152, 2304, 4608, 256,
              512, 1024, 2048, 4096, 8192, 16384, or 32768  (unless  --lax  is
              used)
```
those music that i had are mostly 1920 block and maximum is 4608 block, so i can fixed the buffer used to decode them

##  skipping of other metadata block

To quickly get to decoding of the audio, we skipped the other metadata block in the flac file, these metadata block contains the album images and cover info.

```
    while (1) {
        uint8_t block_header[4];
        fread(block_header, 1, 4, file_in);

        uint8_t last = (block_header[0] & 0x80) >> 7;
        uint8_t type = block_header[0] & 0x7F;
        uint32_t length = (block_header[1] << 16) | (block_header[2] << 8) | (block_header[3]);

        if (type == 0) {
            printf("shouldnt be\n");
        } else {
            fseek(file_in, length, SEEK_CUR);
            printf("skipped a metadata block, its size: %d\n", length);
        }

        if (last) {
            break;
        }
    }
```

Reading the most significant bit in block_header[0] shows whether its the last metadata, reading its length to know how many bytes to skip forward.

## prepare wav file header

To store the pcm output decoded from the flac file, we can write the raw pcm value into a .wav file, written by llm.

```
    // open output file
    FILE *file_out = fopen("output.wav", "wb");

    // wav header - we will fill size fields after we know total data size
    // write placeholder header first
    uint32_t data_size = num_of_samples * num_channel * (bit_depth / 8);

    // RIFF chunk
    fwrite("RIFF", 1, 4, file_out);
    uint32_t riff_size = data_size + 36;
    fwrite(&riff_size, 4, 1, file_out);
    fwrite("WAVE", 1, 4, file_out);

    // fmt chunk
    fwrite("fmt ", 1, 4, file_out);
    uint32_t fmt_size = 16;
    fwrite(&fmt_size, 4, 1, file_out);

    uint16_t audio_format = 1;
    fwrite(&audio_format, 2, 1, file_out);

    uint16_t num_channel_16 = num_channel;      // ← must be 2 bytes
    fwrite(&num_channel_16, 2, 1, file_out);

    fwrite(&sample_rate, 4, 1, file_out);

    uint32_t byte_rate = sample_rate * num_channel * (bit_depth / 8);
    fwrite(&byte_rate, 4, 1, file_out);

    uint16_t block_align = num_channel * (bit_depth / 8);
    fwrite(&block_align, 2, 1, file_out);

    uint16_t bit_depth_16 = bit_depth;          // ← must be 2 bytes
    fwrite(&bit_depth_16, 2, 1, file_out);

    // data chunk
    fwrite("data", 1, 4, file_out);
    fwrite(&data_size, 4, 1, file_out);
```

## decoding the audio data

flac encodes the audio data using constant, lpc, fixed prediction and verbatim. But in my music collection, every audio only used constant and lpc, I don't find any music in my library that uses fixed prediction and verbatim method, so I will skip implementing it.

* every subframe starts with 0xFF 0xF8, so we can verify that

```
    uint8_t sync_frame[2];
        
    if (fread(sync_frame, sizeof(uint8_t), 2, file_in) < 2) {
        printf("end of file\n");
        break;
    }

    printf("sync frame: 0x%02x 0x%02x\n", sync_frame[0], sync_frame[1]);
    if (sync_frame[0] != 0xff || sync_frame[1] != 0xF8) {
        printf("sync frame wrong!\n");
        break;
    }
```

* parsing the header

```
    // block size, sample rate
    uint8_t byte3;
    fread(&byte3, 1, 1, file_in);
    uint8_t block_size_code = (byte3 >> 4) & 0x0F;
    printf("block size bits: 0b%04b\n", block_size_code);
    uint8_t sample_rate_bits = byte3 & 0x0F;
    printf("sample rate bits: 0b%04b\n", sample_rate_bits);

    uint8_t byte4;
    fread(&byte4, 1, 1, file_in);
    uint8_t chan_asgn = (byte4 >> 4) & 0x0F;
    printf("channel bits: 0b%04b\n", chan_asgn); // 0001 for 2 channel, left and right
    uint8_t bit_depth_bits = (byte4 >> 1) & 0x07;
    printf("bit depth bits: 0b%03b\n", bit_depth_bits); // 110 for 24 bits per sample

    uint8_t utf8_first;
    fread(&utf8_first, 1, 1, file_in);
    int extra = 0;
    if      ((utf8_first & 0x80) == 0x00) extra = 0;
    else if ((utf8_first & 0xE0) == 0xC0) extra = 1;
    else if ((utf8_first & 0xF0) == 0xE0) extra = 2;
    else if ((utf8_first & 0xF8) == 0xF0) extra = 3;
    else if ((utf8_first & 0xFC) == 0xF8) extra = 4;
    else if ((utf8_first & 0xFE) == 0xFC) extra = 5;
    else                                   extra = 6;
    fseek(file_in, extra, SEEK_CUR);
    printf("skipped %d utf8 coded frame\n", extra);

    uint32_t block_size;
    if (block_size_code == 1) {
        block_size = 192;
    } else if (block_size_code >= 2 && block_size_code <= 5) {
        block_size = 576 << (block_size_code - 2);
    } else if (block_size_code == 6) {
        uint8_t bs;
        fread(&bs, 1, 1, file_in);
        block_size = bs + 1;
    } else if (block_size_code == 7) {
        uint8_t bs[2];
        fread(&bs, 1, 2, file_in);
        block_size = ((bs[0] << 8) | bs[1]) + 1;
    } else if (block_size_code >= 8 && block_size_code <= 15) {
        block_size = 256 << (block_size_code - 8);
    }
    printf("block size: %d samples\n", block_size);

    if (sample_rate_bits == 12) {
        fseek(file_in, 1, SEEK_CUR);
    } else if (sample_rate_bits == 13 || sample_rate_bits == 14) {
        fseek(file_in, 2, SEEK_CUR);
    }

    fseek(file_in, 1, SEEK_CUR); // skip crc8
```

## fixme add the header for frame later

## constant value

* read a single value with the bitsize of eff_depth, then fill the whole block with the same value, usually the value is 0 when the subframe is silence

```
    if (subframe_type == 0) {
        // constant
        num_const_subframe++;
        int32_t val = read_signed_bits(&br, eff_depth);
        printf("constant value: %d\n", val);
        for (int i = 0; i < block_size; i++)
            result_ch[ch][i] = val << wasted_bits;
```

## LPC decoding

* we get the order number first, order of 1 means it only uses the previous value, order of 2 means it uses the value[t-1], value[t-2] to predict the value[t]

```
    else if (subframe_type >= 32 && subframe_type <= 63) {
        // lpc
        num_lpc_subframe++;
        uint8_t order = subframe_type - 31;
        printf("lpc order: %d\n", order);
```

* warmup values is the first few values, they are stored in raw directly, with the side of eff_depth

```
        // read warmup values
        for (int i = 0; i < order; i++)
            result_ch[ch][i] = read_signed_bits(&br, eff_depth);
```

* now read precision, shift and coefficients, precision is just the length of coefficients

```
        int precision = read_bits(&br, 4) + 1;
        int shift = read_signed_bits(&br, 5);
        printf("  precision=%d shift=%d\n", precision, shift);

        // read coefficients
        int32_t coefs[32];
        for (int i = 0; i < order; i++) {
            coefs[i] = read_signed_bits(&br, precision);
            printf("  coef[%d] = %d\n", i, coefs[i]);
        }
```

* read the coding method parameters, the subframe is further divided into multiple partitions

```
        // read coding method
        int32_t *residuals = malloc(block_size * sizeof(int32_t));
        int method = read_bits(&br, 2); // 0 for 4 bit rice param, 1 for 5 bit rice param
        int param_bits = (method == 0) ? 4 : 5;
        int escape_code = (method == 0) ? 0xF : 0x1F;
        printf("  residual method=%d\n", method);

        // partition order
        int partition_order = read_bits(&br, 4);
        int num_partitions = 1 << partition_order;
        printf("  partition_order=%d num_partitions=%d\n", partition_order, num_partitions);
```

* partition size = block size / num partitions (block_size >> partition order is just optimisation), for the first p=0, it start at 0*size + order, end at 1x side

```
p=0: start = 0*size + order,  end = 1*size
p=1: start = 1*size + 0,      end = 2*size
p=2: start = 2*size + 0,      end = 3*size
```

```
        for (int p = 0; p < num_partitions; p++) {
            // first partition excludes warm-up samples
            int start = p * (block_size >> partition_order) + (p == 0 ? order : 0);
            int end   = (p + 1) * (block_size >> partition_order);

```
* it can either store samples in raw, or in rice coded
```
            int param = read_bits(&br, param_bits);

            if (param == escape_code) {
                // escaped: samples stored raw
                int num_bits = read_bits(&br, 5);
                for (int i = start; i < end; i++)
                    residuals[i] = read_signed_bits(&br, num_bits);
            } else {
                // rice coded
                for (int i = start; i < end; i++) {
                    // count zeros = quotient
                    uint32_t quotient = 0;
                    while (read_bits(&br, 1) == 0)
                        quotient++;
                    // read param bits = remainder
                    uint32_t remainder = read_bits(&br, param);
                    // combine
                    uint32_t folded = (quotient << param) | remainder;
                    // zigzag decode
                    residuals[i] = (folded & 1) ? -(int32_t)(folded >> 1) - 1
                                                :  (int32_t)(folded >> 1);
                }
            }
        }
```

### LPC reconstruction

* multiply previous value with coefficient, loop through all the order, save it in result_ch

```
        for (int i = order; i < block_size; i++) {
            int64_t sum = 0;
            for (int j = 0; j < order; j++)
                sum += (int64_t)coefs[j] * result_ch[ch][i - 1 - j];
            result_ch[ch][i] = residuals[i] + (int32_t)(sum >> shift);
        }
```

* for example, if the audio is 24 bit, but it only used 20 bits of real data, then wasted bits is 4

```
        for (int i = 0; i < block_size; i++)
            result_ch[ch][i] <<= wasted_bits;

        free(residuals);
```


## decorrelation

* first check the chan_asgn value, it shows how the audio channel is encoded
* for chan_asgn == 8, ch0 is left channel, ch1 is right channel, the result_ch[1] is actually storing the difference
* chan_asgn == 9 is storing right channel in result_ch1, and the difference is result_ch0
* chan_asgn == 10 is storing the average and side difference, it do right = mid - side /2, left = left + side

```
        // decorrelation
        if (chan_asgn == 8) {
            for (int i = 0; i < block_size; i++)
                result_ch[1][i] = result_ch[0][i] - result_ch[1][i];
        } else if (chan_asgn == 9) {
            for (int i = 0; i < block_size; i++)
                result_ch[0][i] += result_ch[1][i];
        } else if (chan_asgn == 10) {
            for (int i = 0; i < block_size; i++) {
                int64_t mid   = result_ch[0][i];
                int64_t side  = result_ch[1][i];
                int64_t right = mid - (side >> 1);
                int64_t left  = right + side;
                result_ch[0][i] = (int32_t)left;
                result_ch[1][i] = (int32_t)right;
            }
        }
```

* writing output in wav file, only write 2 byte for 16 bit, 3 byte for 24 bit

```
        for (int i = 0; i < block_size; i++) {
            for (int ch = 0; ch < num_channel; ch++) {
                int32_t val = result_ch[ch][i];
                // write bit_depth/8 bytes, little endian
                fwrite(&val, 1, bit_depth / 8, file_out);
            }
        }

        // free
        for (int ch = 0; ch < num_channel; ch++) {
            free(result_ch[ch]);
            result_ch[ch] = NULL;
        }

        if (hit_new_type) break;
```

* also, it will do byte align in the ending, and 2 byte for crc, just skip them and we can continue decode another block

```
        // byte align
        br.bits_left -= br.bits_left % 8;

        read_bits(&br, 16); // skip crc
    }
```

