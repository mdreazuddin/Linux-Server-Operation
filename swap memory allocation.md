# Create a Swap File in production

## create swap memory in Ubuntu 22.04 using a swap file.

**Step 1: Check if Swap Exists**

`swapon --show`

If no output is shown, it means there is no active swap.

**Step 2: Create a Swap File for 8GB memory**

`sudo fallocate -l 8G /swapfile`

**Step 3: Set Correct Permissions for root user**

`sudo chmod 600 /swapfile`

**Step 4: Convert the File to Swap Space**

`sudo mkswap /swapfile`

**Step 5: Enable the Swap File**

`sudo swapon /swapfile`

To verify run the command:

`swapon --show`

**Step 6: Make Swap Permanent add it to /etc/fstab**

/swapfile none swap sw 0 0
