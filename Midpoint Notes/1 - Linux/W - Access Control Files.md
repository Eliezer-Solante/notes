![[Pasted image 20260715194519.png]]
![[Pasted image 20260715194651.png]]
![[Pasted image 20260715194929.png]]
![[Pasted image 20260715195009.png]]


Access Control Lists (ACLs) in Linux give you **fine-grained control** over file permissions beyond the basic owner/group/others model. They let you assign specific rights to individual users or groups. Here’s a clear breakdown of the key ACL commands:

## 🔑 Core ACL Commands

- `setfacl` → Set or modify ACLs on a file/directory.
    - Add/modify a user’s permission:
        bash
        ```
        setfacl -m u:alice:rw file.txt
        ```
        → Grants `alice` read/write.

    - Add/modify a group’s permission:
        bash        
        ```
        setfacl -m g:developers:r file.txt
        ```
        
        → Grants group `developers` read-only.
        
    - Remove an ACL entry:       
        bash
        
        ```
        setfacl -x u:alice file.txt
        ```       
        → Deletes `alice`’s custom ACL.
        
- `getfacl` → Display ACLs.  
    bash    
    ```
    getfacl file.txt
    ```    
    → Shows current owner/group permissions plus any ACL entries.
    
- **Default ACLs** (for directories):    
    - Apply ACLs to all new files created inside:        
        bash      
        ```
        setfacl -m d:u:alice:rw /project
        ```        
       → Any new file in `/project` gives `alice` read/write automatically.
        
- **Recursive ACLs**:    
    - Apply ACLs to all files/directories inside:        
        bash      
        ```
        setfacl -R -m u:alice:rw /project
        ```
        
- **Remove all ACLs**:   
    bash   
    ```
    setfacl -b file.txt
    ```  
    → Clears all ACL entries, leaving only standard owner/group/others permissions.