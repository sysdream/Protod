#!/usr/bin/python

"""
Protod, version 1.1 - Generic version

Copyright (c) 2012 SYSDREAM


Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell 
copies of the Software, and to permit persons to whom the Software is furnished
to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED,
INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A
PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF
CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE
OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

Author: Damien Cauquil <d.cauquil@sysdream.com>

"""

import sys
import os

# require google's protobuf library
from google.protobuf.descriptor_pb2 import FileDescriptorProto, FieldDescriptorProto
from google.protobuf.message import DecodeError


###########
# helpers
###########

def is_valid_filename(filename):
    '''
    Check if given filename may be valid
    '''
    charset = 'abcdefghijklmnopqrstuvwxyz0123456789-_/$,.[]()'
    for char in filename.lower():
        if char not in charset:
            return False
    return True


def decode_varint128(stream):
    '''
    Decode Varint128 from buffer
    '''
    bits = ''
    count = 0
    for stream_byte in stream:
        count += 1
        raw_byte = ord(stream_byte)
        bits += (bin((raw_byte&0x7F))[2:]).rjust(7,'0')
        if (raw_byte&0x80) != 0x80:
            break
    return (int(bits, 2), count)


def render_type(field_type, package):
    '''
    Return the string representing a given type inside a given package
    '''
    i = 0
    nodes = field_type.split('.')
    nodes_ = package.split('.')
    for i in range(len(nodes)):
        if i < len(nodes_):
            if nodes[i] != nodes_[i]:
                return '.'.join(nodes[i:])
        else:
            return '.'.join(nodes[i:])
    return '.'.join(nodes[i:])


#############################
# Protobuf fields walker
#############################

class ProtobufFieldsWalker:
    '''
    Homemade Protobuf fields walker
    
    This class allows Protod to walk the fields
    and determine the probable size of the protobuf
    serialized file.
    '''
    
    def __init__(self, stream):
        self._stream = stream
        self._size =  -1
    
    def get_size(self):
        return self._size
    
    def walk(self):
        end = False
        offset = 0
        while (not end) and (offset<len(self._stream)):
            # read tag
            tag = ord(self._stream[offset])
            offset += 1
            wire_type = tag&0x7
            if wire_type == 0:
                value, size = decode_varint128(self._stream[offset:])
                offset += size
            elif wire_type == 1:
                offset += 8
            elif wire_type == 2:
                value, size = decode_varint128(self._stream[offset:])
                offset += size + value
            elif wire_type == 5:
                offset += 4
            elif wire_type == 3:
                continue
            elif wire_type == 4:
                continue
            else:
                end = True
        self._size = offset-1


#############################
# Serialized metadata parsing
#############################

class FileDescriptorDisassembler:
    '''
    Core disassembling class
    
    This class parses the provided serialized data and
    produces one or many .proto files.
    '''
    
    def __init__(self, file_desc):
        self.desc = file_desc

    def getLabel(self, l):
        return [None, 'optional', 'required', 'repeated'][l]

    def getTypeStr(self, t):
        types = [
            None,
            'double',
            'float',
            'int64',
            'uint64',
            'int32',
            'fixed64',
            'fixed32',
            'bool',
            'string',
            'group',
            'message',
            'bytes',
            'uint32',
            'enum',
            'sfixed32',
            'sfixed64',
            'sint32',
            'sint64'
        ]
        return types[t]
    
    def renderEnum(self, enum, depth=0, package='', nested=False):
        buffer = '\n'
        buffer += '%senum %s {\n' % (' '*depth, enum.name)
        for value in enum.value:
            buffer += '%s%s = %d;\n' % (' '*(depth+1), value.name, value.number)
        buffer += '%s}' % (' '*depth)
        buffer += '\n'
        return buffer

    def renderField(self, field, depth=0, package='', nested=False):
        buffer = ''
        try:
            if field.HasField('type'):
                # message case
                if field.type == FieldDescriptorProto.TYPE_MESSAGE or field.type == FieldDescriptorProto.TYPE_ENUM:
                    field.type_name = render_type(field.type_name[1:], package)
                    buffer += '%s%s %s %s = %d;\n' % (' '*depth, self.getLabel(field.label), field.type_name, field.name, field.number)
                else:
                    if field.HasField('default_value'):
                        if self.getTypeStr(field.type) == 'string':
                            field.default_value = '"%s"'% field.default_value
                        buffer += '%s%s %s %s = %d [default = %s];\n' % (' '*depth, self.getLabel(field.label), self.getTypeStr(field.type), field.name, field.number, field.default_value)
                    else:    
                        buffer += '%s%s %s %s = %d;\n' % (' '*depth, self.getLabel(field.label), self.getTypeStr(field.type), field.name, field.number)
        except ValueError:
            buffer += '%smessage %s {\n' % (' '*depth, field.name)
            _package = package+'.'+field.name
            
            if len(field.nested_type)>0:
                for nested in field.nested_type:
                    buffer += self.renderField(nested, depth+1, _package, nested=True)
            if len(field.enum_type)>0:
                for enum in field.enum_type:
                    buffer += self.renderEnum(enum, depth+1, _package)
            if len(field.field)>0:
                for field in field.field:
                    buffer += self.renderField(field, depth+1, _package)
            buffer += '%s}' % (' '*depth)
            buffer += '\n\n'
        return buffer


    def render(self, filename=None):
        print '[+] Processing %s' % self.desc.name
        buffer = ''
        buffer += 'package %s;\n\n' % self.desc.package
        
        # add dependencies
        if len(self.desc.dependency)>0:
            for dependency in self.desc.dependency:
                buffer += 'import "%s";\n' % dependency
            buffer += '\n'
        
        if len(self.desc.enum_type)>0:
            for enum in self.desc.enum_type:
                buffer += self.renderEnum(enum, package=self.desc.package)
        if len(self.desc.message_type)>0:
            for message in self.desc.message_type:
                buffer += self.renderField(message, package=self.desc.package)
        if filename:
            _dir = os.path.dirname(filename)
            if _dir != '' and not os.path.exists(_dir):
                os.makedirs(_dir)
            open(filename,'w').write(buffer)
        else:        
            _dir = os.path.dirname(self.desc.name)
            if _dir != '' and not os.path.exists(_dir):
                os.makedirs(_dir)
            open(self.desc.name,'w').write(buffer)

#############################
# Main code
#############################

class ProtobufExtractor:
    def __init__(self, filename=None):
        self.filename = filename
    
    def extract(self):
        try:
            content = open(self.filename,'rb').read()

            # search all '.proto' strings
            protos = []
            stream = content
            while len(stream)>0:
                try:
                    r = stream.index('.proto')
                    for j in range(64):
                        try:
                            if decode_varint128(stream[r-j:])[0]==(j+5) and is_valid_filename(stream[r-j+1:r+6]):
                                # Walk the fields and get a probable size
                                walker = ProtobufFieldsWalker(stream[r-j-1:])
                                walker.walk()
                                probable_size = walker.get_size()
                                
                                """
                                Probable size approach is not perfect,
                                we add a delta of 1024 bytes to be sure
                                not to miss something =)
                                """
                                for k in range(probable_size+1024, 0, -1):
                                    try:
                                        fds  = FileDescriptorProto()
                                        fds.ParseFromString(stream[r-j-1:r-j-1+k])
                                        protos.append(stream[r-j-1:r-j-1+k])
                                        print '[i] Found protofile %s (%d bytes)' % (stream[r-j+1:r+6], k)
                                        break
                                    except DecodeError:
                                        pass
                                    except UnicodeDecodeError:
                                        pass
                                break
                        except IndexError:
                            pass
                    stream = stream[r+6:]
                except ValueError:
                    break

            # Load successively each binary proto file and rebuild it from scratch
            seen = []
            for content in protos:
                try:
                    # Load the prototype
                    fds  = FileDescriptorProto()
                    fds.ParseFromString(content)
                    res = FileDescriptorDisassembler(fds)
                    if len(res.desc.name)>0:
                        if res.desc.name not in seen:
                            open(res.desc.name+'.protoc','wb').write(content)
                            res.render()
                            seen.append(res.desc.name)
                except DecodeError:
                    pass
            
        except IOError:
            print '[!] Unable to read %s' % sys.argv[1]        

if __name__ == '__main__':
    if len(sys.argv)>=2:
        print "[i] Extracting from %s ..." % sys.argv[1]
        extractor = ProtobufExtractor(sys.argv[1])
        extractor.extract()
        print "[i] Done"
    else:
        print "[ Protod (Protobuf metadata extractor) (c) 2012 Sysdream  ]"
        print ''
        print '[i] Usage: %s [executable]' % sys.argv[0]
