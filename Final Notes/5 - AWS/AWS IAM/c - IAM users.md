![[Pasted image 20260805104438.png]]
![[Pasted image 20260805104419.png]]

![[Pasted image 20260805104532.png]]

![[Pasted image 20260805104559.png]]

DEMO NOTES

==CloudShell==
![[Pasted image 20260805105220.png]]
displays current user 

To switch to another user
![[Pasted image 20260805105422.png]]
create Access ID on the console

verify the account switch run `aws sts get-caller-identity` again


Additional:
Attach a policy using CLI
ex. attaching a `ReadOnlyAccess` Policy to a IAM user
`aws iam attach-user-policy --user-name iamuser-1 --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess`