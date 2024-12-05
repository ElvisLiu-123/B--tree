 

```c
#include <stdlib.h>
#include <stdio.h>
#include "BPlusTree.h"

// Function to search for the largest index less than or equal to target in the node's keys
long searchKeyInNode(Node *node, long target) {
    long *arr = node->key;
    long low = 0;
    long high = node->n;

    while (low < high) {
        long mid = low + ((high - low) >> 1);
        if (target >= arr[mid]) {
            low = mid + 1;
        } else {
            high = mid;
        }
    }
    return low - 1;
}

// Function to split the child node at index i of parent
void splitNode(Node *parent, long i) {
    Node *child = parent->child[i];
    Node *sibling = createNode(child->isLeaf);
    sibling->right = child->right; // Sibling's right pointer

    if (child->isLeaf) {
        sibling->n = child->n / 2; // Leaf node splitting
        child->n -= sibling->n;     // Update child node key count

        sibling->data = (long*)malloc(sizeof(long) * (MAX_KEY + 1));
        for (long j = 0; j < sibling->n; j++) {
            sibling->key[j] = child->key[child->n + j];  // Copy keys to sibling
            sibling->data[j] = child->data[child->n + j]; // Copy data to sibling
        }
    } else {
        sibling->n = child->n / 2; // Non-leaf node splitting
        child->n -= sibling->n + 1;

        sibling->child = (Node**)malloc(sizeof(Node*) * (MAX_CHILD + 1));
        for (long j = 0; j < sibling->n; j++) {
            sibling->key[j] = child->key[child->n + j + 1]; // Copy keys to sibling
            sibling->child[j] = child->child[child->n + j + 1]; // Copy children to sibling
        }
    }

    for (long j = parent->n; j > i; j--) {
        parent->child[j + 1] = parent->child[j]; // Shift children to the right
        parent->key[j] = parent->key[j - 1];     // Shift keys to the right
    }
    parent->key[i] = child->key[child->n];       // Promote a key to parent
    parent->child[i + 1] = sibling;              // Link sibling
    parent->n++;
}

// Function to insert key,value pair into the node
long insertToNode(Node *node, long key, long value) {
    long current_idx = searchKeyInNode(node, key);
    
    if (node->isLeaf) {
        if (current_idx >= 0 && node->key[current_idx] == key) {
            return -1; // Duplicate key
        }

        for (long j = node->n; j > current_idx + 1; j--) {
            node->key[j] = node->key[j - 1];
            node->data[j] = node->data[j - 1];
        }
        node->key[current_idx + 1] = key;
        node->data[current_idx + 1] = value;
        node->n++;
        return 0; // Successful insertion
    }

    Node *child = node->child[current_idx + 1];
    if (child->n == MAX_KEY) {
        splitNode(node, current_idx + 1);
        if (node->key[current_idx + 1] < key) {
            current_idx++;
        }
    }
    return insertToNode(node->child[current_idx + 1], key, value); // Recur to child
}

// Function to merge child k and k+1 of parent
void mergeNode(Node *parent, long k) {
    Node *leftChild = parent->child[k];
    Node *rightChild = parent->child[k + 1];

    if (leftChild->isLeaf) {
        for (long i = leftChild->n; i < leftChild->n + rightChild->n; i++) {
            leftChild->key[i] = rightChild->key[i - leftChild->n];
            leftChild->data[i] = rightChild->data[i - leftChild->n];
        }
        leftChild->n += rightChild->n;
    } else {
        leftChild->key[leftChild->n] = parent->key[k]; // Promote parent key
        leftChild->n++;
        for (long i = leftChild->n; i < leftChild->n + rightChild->n + 1; i++) {
            leftChild->key[i] = rightChild->key[i - leftChild->n - 1];
            leftChild->child[i] = rightChild->child[i - leftChild->n - 1];
        }
        leftChild->n += rightChild->n + 1; // Including promoted key
    }

    for (long i = k + 1; i < parent->n; i++) {
        parent->key[i - 1] = parent->key[i];
        parent->child[i] = parent->child[i + 1];
    }
    parent->n--;

    free(rightChild);
}

// Function to delete key from node
long delFromNode(Node *node, long key) {
    long current_idx = searchKeyInNode(node, key);
    
    if (node->isLeaf) {
        if (current_idx < 0 || node->key[current_idx] != key) {
            return -1; // Key not found
        }

        for (long j = current_idx; j < node->n - 1; j++) {
            node->key[j] = node->key[j + 1];
            node->data[j] = node->data[j + 1];
        }
        node->n--;
        return 0; // Successful deletion
    }

    Node *child = node->child[current_idx + 1];
    if (delFromNode(child, key) == -1) return -1;

    if (child->n < MIN_KEY) {
        if (current_idx > 0 && node->child[current_idx - 1]->n > MIN_KEY) {
            Node *leftSibling = node->child[current_idx - 1];
            child->key[child->n] = node->key[current_idx - 1];
            child->n++;
            node->key[current_idx - 1] = leftSibling->key[leftSibling->n - 1];
            leftSibling->n--;
        } else if (current_idx + 1 < node->n && node->child[current_idx + 1]->n > MIN_KEY) {
            Node *rightSibling = node->child[current_idx + 1];
            child->key[child->n] = node->key[current_idx];
            child->n++;
            node->key[current_idx] = rightSibling->key[0];
            for (long i = 0; i < rightSibling->n - 1; i++) {
                rightSibling->key[i] = rightSibling->key[i + 1];
            }
            rightSibling->n--;
        } else {
            if (current_idx == node->n) {
                mergeNode(node, current_idx - 1);
            } else {
                mergeNode(node, current_idx);
            }
        }
    }
    return 0; // Deletion successful
}

/*
    External Functions
*/

// Create and initialize the B+ tree.
Tree* createTree() {
    Node *root = createNode(1); // Create a leaf node
    Tree *tree = (Tree*)malloc(sizeof(Tree));
    tree->root = root;
    tree->size = 0;
    return tree;
}

// Search for a key and return the corresponding value, or -1 if not found.
long equalSearch(Tree *tree, long key) {
    Node *r = tree->root;
    while (1) {
        long current_idx = searchKeyInNode(r, key);
        if (r->isLeaf) {
            return (current_idx >= 0 && r->key[current_idx] == key) ? r->data[current_idx] : -1;
        } else {
            r = r->child[current_idx + 1];
        }
    }
}

// Perform a range search for keys in the range [start, end].
long rangeSearch(Tree *tree, long start, long end, KV *buffer, long buffer_length) {
    Node *r = tree->root;
    long current_idx;

    while (1) {
        current_idx = searchKeyInNode(r, start);
        if (r->isLeaf) {
            break; // Found the leaf
        }
        r = r->child[current_idx + 1];
    }

    if (current_idx < 0) { // Adjust if no valid index
        current_idx = 0;
    }

    long count = 0;

    // Collect data in the leaf node
    while (current_idx < r->n && count < buffer_length) {
        if (r->key[current_idx] >= start && r->key[current_idx] <= end) {
            buffer[count].key = r->key[current_idx];
            buffer[count].value = r->data[current_idx];
            count++;
        }
        current_idx++;
    }

    // Traverse to the right leaf nodes
    while (r->right && count < buffer_length) {
        r = r->right;
        for (current_idx = 0; current_idx < r->n && count < buffer_length; current_idx++) {
            if (r->key[current_idx] >= start && r->key[current_idx] <= end) {
                buffer[count].key = r->key[current_idx];
                buffer[count].value = r->data[current_idx];
                count++;
            }
        }
    }

    return count; // Return total found count
}

// Get the height of the tree
long treeHeight(Tree *tree) {
    long height = 1;
    Node *r = tree->root;
    while (!r->isLeaf) {
        height++;
        r = r->child[0];
    }
    return height;
}

long insertToTree(Tree *tree, long key, long value) {
    if (insertToNode(tree->root, key, value) == -1) {
        return -1; // Duplicate key
    }

    // Check for root split
    if (tree->root->n > MAX_KEY) {
        Node *old_root = tree->root;
        Node *new_root = createNode(0);
        new_root->child[0] = old_root; // Set old root as child of the new root
        old_root->parent = new_root;
        
        splitNode(new_root, 0); // Split root
        tree->root = new_root;   // Update root
    }
    tree->size++;
    return 0;
}

long delFromTree(Tree *tree, long key) {
    if (delFromNode(tree->root, key) == -1) {
        return -1; // Key not found
    }

    // Handle case where root needs to be replaced
    if (tree->root->n == 0) {
        Node *old_root = tree->root;
        if (!tree->root->isLeaf) {
            tree->root = tree->root->child[0]; // Promote child as new root
        }
        free(old_root->data);
        free(old_root->child);
        free(old_root);
    }
    tree->size--;
    return 0;
}

/*
    Debugging Functions
*/

// Helper function to recursively print the tree structure.
void printTreeRec(Node *node, long level) {
    printf("level %ld\n", level);
    if (node->isLeaf) {
        for (long i = 0; i < node->n; i++) {
            printf("(%ld, %ld) ", node->key[i], node->data[i]);
        }
        printf("\n");
    } else {
        for (long i = 0; i < node->n; i++) {
            printf("%ld ", node->key[i]);
        }
        printf("\n");
        for (long i = 0; i <= node->n; i++) {
            printTreeRec(node->child[i], level * 10 + i);
        }
    }
}

// Print the structure of the tree.
void printTree(Tree *tree) {
    printf("----------start of tree----------\n");
    printTreeRec(tree->root, 1);
    printf("----------end of tree----------\n");
}
```

