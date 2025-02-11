# Add AP Licenses and Upgrade Entitlement to a ZoneDirector 1200

Ruckus ships ZoneDirector controllers with a license to control a limited number of Access Points.  
Extra APs can be controlled by purchasing and uploading a license file.  
> But Ruckus won't sell you AP licenses unless you are the original purchaser of the ZoneDirector.

Ruckus will prevent you from applying software upgrades to your ZoneDirector unless you have an active Support Contract.  
> But Ruckus won't sell you a Support Contract unless you are the original purchaser of the ZoneDirector.

To prevent e-waste and save these from landfills, you may use the procedure here to enable upgrades and apply free AP licenses to your ZoneDirector for use in a homelab or personal environment.

>If you have a ZoneDirector 1000/1100/3000/5000 then you can still add AP Licenses, but you'll have to (temporarily) install older software.  
>Follow the instructions here: [Add AP Licenses to a ZoneDirector](ZDAddLicenses.md)

>If you also want a root shell, follow the instructions here: [Add a Root Shell to your ZoneDirector 1200](ZD1200AddRootShell.md)

The script below patches any ZD1200 Software Image so that you always have 75 AP licenses and your Upgrade entitlement ends in 2027.  
If you already have the latest software version or you have no active support then just download, patch & apply the version you're currently running.

Save the script below to e.g. `patch_zd_image.sh`, make the script executable (e.g. `chmod +x patch_zd_image.sh`) then you can patch any ZD1200 installation:-
```bash
./patch_zd_image.sh zd1200_10.5.1.0.176.ap_10.5.1.0.176.img patched1051.img
```

I've tested the script on all currently-public ZD1200 releases (9.9 - 10.5) for both web and cli upgrades.

>If you decide to tweak the script, and your upgrade hangs, don't panic!  
>Probably cycling the power will bring you back to the un-upgraded software, where you can have another try.  
>Even if your ZD1200 won't come back to life, you can pull the cfcard out and re-flash with a clean image from your PC.  
>Contact me in Issues if you need me to provide you with an image to flash.

## Patch from Linux or WSL

```bash
#!/bin/bash

function rks_encrypt {
RUCKUS_SRC="$1" RUCKUS_DEST="$2" python3 - <<END
import os
import struct

input_path = os.environ['RUCKUS_SRC']
output_path = os.environ['RUCKUS_DEST']

(xor_int, xor_flip) = struct.unpack('QQ', b')\x1aB\x05\xbd,\xd6\xf25\xad\xb8\xe0?T\xc58')
structInt8 = struct.Struct('Q')

with open(input_path, "rb") as input_file:
    with open(output_path, "wb") as output_file:
        input_len = os.path.getsize(input_path)
        input_blocks = input_len // 8
        output_int = 0
        input_data = input_file.read(input_blocks * 8)
        for input_int in struct.unpack_from(str(input_blocks) + "Q", input_data):
            output_int ^= xor_int ^ input_int
            xor_int ^= xor_flip
            output_file.write(structInt8.pack(output_int))
        
        input_block = input_file.read()
        input_padding = 8 - len(input_block)
        input_int = structInt8.unpack(input_block.ljust(8, bytes([input_padding | input_padding << 4])))[0]
        output_int ^= xor_int ^ input_int
        output_file.write(structInt8.pack(output_int))
END
}
function rks_decrypt {
RUCKUS_SRC="$1" RUCKUS_DEST="$2" python3 - <<END
import os
import struct

input_path = os.environ['RUCKUS_SRC']
output_path = os.environ['RUCKUS_DEST']

(xor_int, xor_flip) = struct.unpack('QQ', b')\x1aB\x05\xbd,\xd6\xf25\xad\xb8\xe0?T\xc58')
structInt8 = struct.Struct('Q')

with open(input_path, "rb") as input_file:
    with open(output_path, "wb") as output_file:
        input_data = input_file.read()
        previous_input_int = 0
        for input_int in struct.unpack_from(str(len(input_data) // 8) + "Q", input_data):
            output_bytes = structInt8.pack(previous_input_int ^ xor_int ^ input_int)
            xor_int ^= xor_flip
            previous_input_int = input_int
            output_file.write(output_bytes)
        
        output_padding = int.from_bytes(output_bytes[-1:], 'big') & 0xf
        output_file.seek(-output_padding, os.SEEK_END)
        output_file.truncate()
END
}

rm -rf zdimage
mkdir -p zdimage

rks_decrypt "$1" zdimage/zd.img.tgz

pushd zdimage

gzip -d zd.img.tgz
tar -xvf zd.img.tar ac_upg.sh
sed -i -e '/echo "FILE:`\/usr\/bin\/md5sum \.\/\$ZD_KERNEL`" >>\/mnt\/file_list\.txt/a \
cd \/mnt\/etc\/persistent-scripts\
cat <<EOF >support\
<support-list>\
	<support zd-serial-number="`cat \/bin\/SERIAL`" service-purchased="904" date-start="1661705940" date-end="1819472340" ap-support-number="licensed" DELETABLE="false"><\/support>\
<\/support-list>\
EOF\
cat <<EOF >license-list\.xml\
<license-list name="75 AP Management" max-ap="75" max-client="4000" value="0x0000000f" urlfiltering-ap-license="0">\
    <license id="1" name="70 AP Management" inc-ap="70" generated-by="264556" serial-number="`cat \/bin\/SERIAL`" status="0" detail="" \/>\
<\/license-list>\
EOF\
tar -czf support\.spt support\
CUR_WRAP_MD5=`md5sum \/mnt\/bin\/sys_wrapper\.sh | cut -d\x27 \x27 -f1`\
sed -i -e \x27\/verify-upload-support)\/a \\\
        cd \\\/tmp\\\
        cat \\\/etc\\\/persistent-scripts\\\/support > support\\\
        cat \\\/etc\\\/persistent-scripts\\\/license-list\\\.xml > \\\/etc\\\/airespider\\\/license-list\\\.xml\\\
        echo "OK"\\\
        ;;\\\
    verify-upload-support-unpatched)\x27 -e \x27\/wget-support-entitlement)\/a \\\
        cat \\\/etc\\\/persistent-scripts\\\/support\\\.spt > "\\\/tmp\\\/$1"\\\
        echo "OK"\\\
        ;;\\\
    wget-support-entitlement-unpatched)\x27 \/mnt\/bin\/sys_wrapper\.sh\
NEW_WRAP_MD5=`md5sum \/mnt\/bin\/sys_wrapper\.sh | cut -d\x27 \x27 -f1`\
sed -i -e "s\/\$CUR_WRAP_MD5\/\$NEW_WRAP_MD5\/" \/mnt\/file_list\.txt' ac_upg.sh
tar uvf zd.img.tar ac_upg.sh
gzip zd.img.tar
popd
rks_encrypt zdimage/zd.img.tar.gz "$2"
```
