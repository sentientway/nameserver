# Release Notes

1. [Msg Interface:](implementation.md#Msg Interface:) nameservice types need to import from Cosmos SDK directory and not the user created nameservice type. I am not sure whether or not this import comes from the ./x/internal directory in the Cosmos SDK.
	
2. I was not able to fully research the CLI. My implementation does not show the creation of nsd or nscli because it seems like the CLI should be created at the app level where nameservice types would import into a CLI routine. It is still not clear to me where the cmd file lives or how it is built.

3. [Keeper](implementation.md#Keeper): The production path for nameservice path is not entirely clear. Nameservice types are on the sdk-tutorial path.

4. Although minor, in some places the term nameservice is used in the path for repository name and yet the module is named *nameserver*. This issue touches several places.

5. I have felt somewhat handicapped by not being able to run the tutorial and look at the finished code myself. It appears as if I should be running a CLI on my repository on GitHub and not on my local machine. I have known the issue is environment.

6. Because of the DRY structure of the code, it feels like the packages fold in on each other so that a linear progression through the nameserver module is challenging. It did not make sense to me to group the code as it is grouped in the tutorial, so I rearranged some code pieces like Msg Type under the Messages heading. There are several other adjustments to make going through a code centric document an easier read.

