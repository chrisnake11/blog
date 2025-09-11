---
title: LoadModel
published: 2025-09-11T18:12:47Z
description: 'LearnOpenGL中，关于Model Loading章节的总结，包括Assimp库的使用，模型文件加载、模型构建和纹理渲染的步骤。'
image: 'https://learnopengl-cn.github.io/img/03/01/assimp_structure.png'
tags: [OpenGL, Computer Graphics, Model Loading]
category: 'Computer Graphics'
draft: false
---

# Model的组成结构

Assimp库会将一个模型文件抽象为一系列的数据结构。主要包括：场景(Scene)、网格(Mesh)、材质(Material)、纹理(Texture)、顶点(Vertex)等。

其中，场景(`Scene`)表示一个完整的局部空间`local space`下的模型，一个场景由多个网格(`Mesh`)和纹理数据`Material`组成。只要能够构建所有的网格(`Mesh`)，再给网格(`Mesh`)绑定上对应的纹理(`Material`)并着色渲染(`Shading and Render`)，就可以得到完整的模型。

## Assimp库介绍

在Assimp中，模型文件被解析为一个`aiScene`对象。`aiScene`包含了多个`aiMesh`对象、`aiMaterial`对象、`aiNode`对象。

`aiMesh`对象表示一个网格，它记录了顶点、法线、材质结构体、平面的顶点索引、材质索引等信息。其中平面的顶点索引中存储了组成该平面的所有顶点在顶点数组中的索引。

通过Mesh中的各种属性，我们可以得到获取渲染模型的最基本的信息。

在`aiMaterial`对象中，存储了材质的各种属性，包括漫反射颜色、镜面反射颜色、环境光颜色、透明度、折射率等信息。材质还可以包含多个纹理，每个纹理都有一个类型（如漫反射纹理、镜面反射纹理、法线纹理等）和一个文件路径。

通过`aiMaterial`对象，我们可以获取材质的各种光照属性和纹理信息，从而在渲染时应用这些材质和纹理。

在`aiNode`对象中，存储了场景中的节点信息。每个节点可以包含多个子节点和多个`aiMesh`的索引。节点还包含一个变换矩阵，用于描述该节点相对于其父节点的变换。

通过`aiNode`对象，我们可以构建场景的层次结构（每个子节点所代表的`Mesh`构成一组），并应用节点的变换矩阵来定位和旋转模型。

```cpp
struct Vertex {
    glm::vec3 Position; // 顶点位置
    glm::vec3 Normal;   // 顶点法线
    glm::vec2 TexCoords; // 纹理坐标
};
struct Texture {
    unsigned int id; // 纹理ID
    string type;     // 纹理类型（diffuse, specular, normal, height）
    string path;     // 纹理路径
};

struct Mesh{
    std::vector<Vertex> vertices; // 顶点数组
    std::vector<glm::vec3> normals; // 法线数组
    std::vector<Texture> textures; // 纹理数组
    std::vector<unsigned int> materialIndices; // 材质索引数组
    std::vector<unsigned int> faces; // 面索引数组
};

// 面的数组：单个面由多个顶点组成
std::vector<std::vector<unsigned int>> faces; // 每个面的顶点索引数组

struct Material {
    GetTexture(type, i, std::string); // 获取当前材质文件的路径
};

struct Node {
    std::string name; // 节点名称
    glm::mat4 transform; // 节点的变换矩阵
    std::vector<unsigned int> meshIndices; // 该节点包含的网格索引
    std::vector<Node> children; // 子节点
};

struct Scene {
    std::vector<Mesh> meshes; // 场景中的所有网格
    std::vector<Material> materials; // 场景中的所有材质
    std::vector<Node> nodes; // 场景中的所有节点
};
```

![LoadModel-2025-09-11-19-50-46](https://learnopengl-cn.github.io/img/03/01/assimp_structure.png)

[LearnOpenGL](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/)中的assimp结构图。

## 使用Assimp加载模型的步骤

有了Assimp库，结合上面的数据结构分析，我们可以大概了解加载模型的流程：
1. **读取模型文件，获取一个Scene的对象**。使用`Assimp::Importer`类的`ReadFile`方法来加载模型文件，并返回一个`aiScene`对象。
2. **读取`Scene`中的每个节点**。根据树形结构，层次遍历Scene中的Node节点，提取每个节点的Mesh索引。以获取每个Mesh对象。
3. **构造`Mesh`对象，读取文件数据**。从Mesh对象中提取出顶点数据、法线数据、纹理数据对象、材质索引、Face索引。
   1. 其中，材质数据是作为UV坐标的图像存储的，因此需要从Materials数据结构中读取到对应的图像路径，然后直接解析图像得到。
   2. 每次读取材质文件图像时，使用`stb_image.h`库解析图像信息，然后直接将材质拷贝到GPU显存中，返回对应的`buffer id`。
4. **使用Mesh的顶点和向量数据，构建网格**。`Mesh`对象读取完毕后，构造Mesh对象并且创建对应的`VAO、VBO、EBO`，将`Mesh`中的顶点、法线、材质坐标数据拷贝到GPU中。随后，在`vertex shader`中读取顶点数据，构建网格。
5. **渲染网格的纹理**。读取Mesh的Texture数组中的所有纹理信息，并将其绑定到相应的纹理单元上。最后将纹理数据发送给`frag shader`着色器中的`uniform Sampler2D`参数，进行网格着色。

因此我们可以构建出两种对象，分别为`Model`和`Mesh`.

`Model`主要负责前面3个步骤，即：模型文件读取为Scene，处理Scene中的Node节点，读取Mesh以及纹理图像文件中的信息，构建出`Mesh`网格对象。

`Mesh`负责后两个部分，即：读取Mesh中的顶点、法线、纹理坐标数据，调用`vertex shader`着色器，构建网格；读取Texture数组中的纹理信息，调用`frag shader`着色器，渲染网格的纹理信息。

并且，`Mesh`会作为`Model`的成员变量，由`Model`来调用`Mesh`中的构建网格和渲染网格的相关接口。

### 代码

#### Mesh 网格类

```cpp
//
// Created by zyx on 2025/9/11.
//

#ifndef MAIN_MESH_H
#define MAIN_MESH_H
#include <string>
#include <vector>
#include "Shader.h"
#include <glm/glm.hpp>

// vertex data and texture uv coordinates
struct Vertex {
    glm::vec3 position;
    glm::vec3 normal;
    glm::vec2 texCoords; // (u, v) texture coordinates
};

// texture picture data
struct Texture {
    unsigned int id;
    std::string type;
    std::string path; // for texture deduplication
};

// a mesh is a collection of vertices, indices and textures
// each vertex bind to a texture uv coordinate
class Mesh {
public:
    std::vector<Vertex> m_vertices;
    std::vector<unsigned int> m_indices;
    std::vector<Texture> m_textures;

    Mesh(std::vector<Vertex> vertices, std::vector<unsigned int> indices, std::vector<Texture> textures);

    void draw(const Shader &shader) const;
private:
    unsigned int m_VAO{}, m_VBO{}, m_EBO{};
    void setupMesh();
};

#endif //MAIN_MESH_H
```

```cpp
//
// Created by zyx on 2025/9/11.
//

#include "Mesh.h"
#include <glad/glad.h>

Mesh::Mesh(std::vector<Vertex> vertices, std::vector<unsigned int> indices, std::vector<Texture> textures) :
    m_vertices(std::move(vertices)), m_indices(std::move(indices)), m_textures(std::move(textures)) {
    setupMesh();
}

void Mesh::setupMesh() {
    // create buffers/arrays
    glGenVertexArrays(1, &m_VAO);
    glGenBuffers(1, &m_VBO);
    glGenBuffers(1, &m_EBO);

    glBindVertexArray(m_VAO);
    // copy buffer data to gpu
    glBindBuffer(GL_ARRAY_BUFFER, m_VBO);
    glBufferData(GL_ARRAY_BUFFER, m_vertices.size() * sizeof(m_vertices[0]), &m_vertices[0], GL_STATIC_DRAW);

    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, m_indices.size() * sizeof(m_indices[0]), &m_indices[0], GL_STATIC_DRAW);

    // bind vertex attributes to buffer data
    glEnableVertexAttribArray(0);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), static_cast<void *>(nullptr));
    glEnableVertexAttribArray(1);
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), reinterpret_cast<void *>(offsetof(Vertex, normal)));
    glEnableVertexAttribArray(2);
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), reinterpret_cast<void *>(offsetof(Vertex, texCoords)));
}

void Mesh::draw(const Shader &shader) const {
    unsigned int diffuseNr = 1;
    unsigned int specularNr = 1;
    unsigned int normalNr = 1;
    unsigned int heightNr = 1;
    for(unsigned int i = 0; i < m_textures.size(); i++) {
        glActiveTexture(GL_TEXTURE0 + i); // active proper texture unit before binding
        // retrieve texture number (the N in diffuse_textureN)
        std::string number{};
        std::string name = m_textures[i].type;
        if(name == "texture_diffuse")
            number = std::to_string(diffuseNr++);
        else if (name == "texture_specular")
            number = std::to_string(specularNr++);
        else if (name == "texture_normal")
            number = std::to_string(normalNr++);
        else if (name == "texture_height")
            number = std::to_string(heightNr++);
        // now set the sampler to the correct texture unit
        name += number;
        // set uniform sampler to correct texture unit
        shader.setInt(name, i);
        // bind the texture
        glBindTexture(GL_TEXTURE_2D, m_textures[i].id);
    }

    // draw the mesh(all the triangles)
    glBindVertexArray(m_VAO);
    // draw all the
    glDrawElements(GL_TRIANGLES, static_cast<GLsizei>(m_indices.size()), GL_UNSIGNED_INT, nullptr);
    glBindVertexArray(0);
    glActiveTexture(GL_TEXTURE0);
}
```

#### Model 模型类

```cpp
//
// Created by zyx on 2025/9/11.
//
#ifndef MAIN_MODEL_H
#define MAIN_MODEL_H
#include <string>
#include <unordered_map>
#include <vector>
#include "Mesh.h"
#include "Shader.h"
#include <assimp/Importer.hpp>
#include <assimp/scene.h>

class Model {
public:
    explicit Model(const std::string &path, bool gamma = false);
    void draw(const Shader& shader) const;
private:
    std::vector<Mesh> m_meshes;
    std::string m_directory;
    std::unordered_map<std::string, Texture> m_texture_loaded;
    bool gammaCorrection;
    void loadModel(const std::string &path);
    void processNode(const aiNode *node, const aiScene *scene);
    Mesh processMesh(const aiMesh *mesh, const aiScene *scene);
    std::vector<Texture> loadMaterialTextures(const aiMaterial *mat, const aiTextureType type, const std::string &typeName);
};
#endif //MAIN_MODEL_H
```

```cpp
//
// Created by zyx on 2025/9/11.
//
#include "Model.h"
#include <iostream>
#include <glm/glm.hpp>
#include <assimp/postprocess.h>
#include "stb_image.h"
#include <glad/glad.h>
#include <glfw/glfw3.h>

unsigned int TextureFromFile(const char *path, const std::string &directory, bool gamma = false);

Model::Model(const std::string &path, bool gamma) : gammaCorrection(gamma){
    loadModel(path);
}

void Model::draw(const Shader &shader) const {
    for (const auto &mesh : m_meshes) {
        mesh.draw(shader);
    }
}

void Model::loadModel(const std::string &path) {
    // call import to read the model file
    Assimp::Importer importer;
    // flags: triangulate the model, flip the texture coordinates on y-axis
    const aiScene * scene = nullptr;
    scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_GenSmoothNormals | aiProcess_FlipUVs | aiProcess_CalcTangentSpace);

    // check for errors
    // read file failed, incomplete scene or no root node
    if (scene == nullptr || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) {
        std::cout << "ERROR::ASSIMP::" << importer.GetErrorString() << std::endl;
        return;
    }
    // get the directory path of the filepath
    m_directory = path.substr(0, path.find_last_of('/'));
    // process the root node recursively
    processNode(scene->mRootNode, scene);
}

void Model::processNode(const aiNode *node, const aiScene *scene) {
    for (unsigned int i = 0; i < node->mNumMeshes; i++) {
        const aiMesh * mesh = scene->mMeshes[node->mMeshes[i]];
        m_meshes.push_back(processMesh(mesh, scene));
    }

    // recursively process each child node
    for (unsigned int i = 0; i < node->mNumChildren; i++) {
        processNode(node->mChildren[i], scene);
    }
}

Mesh Model::processMesh(const aiMesh *mesh, const aiScene *scene) {
    std::vector<Vertex> vertices;
    std::vector<unsigned int> indices;
    std::vector<Texture> textures;

    // load vertices
    for (unsigned int i = 0; i < mesh->mNumVertices; i++) {
        Vertex vertex{};
        glm::vec3 vector;
        vector.x = mesh->mVertices[i].x;
        vector.y = mesh->mVertices[i].y;
        vector.z = mesh->mVertices[i].z;
        vertex.position = vector;

        if (mesh->HasNormals()) {
            vector.x = mesh->mNormals[i].x;
            vector.y = mesh->mNormals[i].y;
            vector.z = mesh->mNormals[i].z;
            vertex.normal = vector;
        }

        // load first set of texture coordinates only
        if (mesh->mTextureCoords[0]) {
            glm::vec2 vec;
            vec.x = mesh->mTextureCoords[0][i].x;
            vec.y = mesh->mTextureCoords[0][i].y;
            vertex.texCoords = vec;
        }
        else {
            vertex.texCoords = glm::vec2(0.0f, 0.0f);
        }

        vertices.push_back(vertex);
    }

    // load indices
    for (unsigned int i = 0; i < mesh->mNumFaces; i++) {
        aiFace face = mesh->mFaces[i];
        for (unsigned int j = 0; j < face.mNumIndices; j++) {
            indices.push_back(face.mIndices[j]);
        }
    }

    // process materials
    // load material from scene
    const aiMaterial *material = scene->mMaterials[mesh->mMaterialIndex];
    // diffuse maps
    std::vector<Texture> diffuseMaps = loadMaterialTextures(material, aiTextureType_DIFFUSE, "texture_diffuse");
    textures.insert(textures.end(), diffuseMaps.begin(), diffuseMaps.end());
    // specular maps
    std::vector<Texture> specularMaps = loadMaterialTextures(material, aiTextureType_SPECULAR, "texture_specular");
    textures.insert(textures.end(), specularMaps.begin(), specularMaps.end());

    std::vector<Texture> normalMaps = loadMaterialTextures(material, aiTextureType_HEIGHT, "texture_normal");
    textures.insert(textures.end(), normalMaps.begin(), normalMaps.end());

    std::vector<Texture> heightMaps = loadMaterialTextures(material, aiTextureType_AMBIENT, "texture_height");
    textures.insert(textures.end(), heightMaps.begin(), heightMaps.end());

    return Mesh{vertices, indices, textures};
}

std::vector<Texture> Model::loadMaterialTextures(const aiMaterial *mat, const aiTextureType type,
    const std::string &typeName) {
    std::vector<Texture> textures;
    // load each texture from materials
    bool m_skit = false;
    for (unsigned int i = 0; i < mat->GetTextureCount(type); i++) {
        aiString path; // get texture path
        mat->GetTexture(type, i, &path);
        bool skip = false;
        // check texture is loaded before
        if (m_texture_loaded.count(path.C_Str())) {
            textures.push_back(m_texture_loaded[path.C_Str()]);
            skip = true;
        }
        if (!skip) {
            Texture texture{};
            texture.id = TextureFromFile(path.C_Str(), m_directory);
            texture.type = typeName;
            texture.path = path.C_Str();
            textures.push_back(texture);
            // store it as loaded
            m_texture_loaded[path.C_Str()] = texture;
        }
    }
    return textures;
}

unsigned int TextureFromFile(const char* path, const std::string& directory, bool gamma) {
    std::string filename = path;
    filename = directory + "/" + filename;

    unsigned int texture;
    glGenTextures(1, &texture);

    int width{}, height{}, nrComponents{};
    // flip the texture on y-axis
    stbi_set_flip_vertically_on_load(true);
    unsigned char* data = stbi_load(filename.c_str(), &width, &height, &nrComponents, 0);
    if (!data) {
        std::cout << "ERROR: stbi_load failed: " << path << " reason=" << stbi_failure_reason() << std::endl;
        stbi_image_free(data);
        return texture;
    }

    GLenum format;
    if (nrComponents == 1) format = GL_RED;
    else if (nrComponents == 3) format = GL_RGB;
    else if (nrComponents == 4) format = GL_RGBA;


    glBindTexture(GL_TEXTURE_2D, texture);
    glTexImage2D(GL_TEXTURE_2D, 0, format, width, height, 0, format, GL_UNSIGNED_BYTE, data);
    glGenerateMipmap(GL_TEXTURE_2D);

    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    stbi_image_free(data);
    return texture;
}
```


# 参考资料

+ [LearnOpenGL CN Assimp章节](https://learnopengl-cn.github.io/03%20Model%20Loading/01%20Assimp/)
