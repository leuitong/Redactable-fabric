# Redactable Fabric 
The first redactable blockchain based on Hyperlegder Fabric, version 2.1.1. Only users (regulators) who have the corresponding attribute key (satisfy the access policy) can perform deleting/modifying operations. This project provides a new solution for the supervision of the consortium blockchain.

## The Modification of Fabric Source Code
Below we explain our modifications in more details.
### Data Structure
We define a new struct named `Chamhash`, then insert into `Payload` and `ProposalResponsePayload` fields:

```go
type Chamhash struct {
	HashValue            []byte   
	RandomValue          []byte   
	Etdcipher            []byte
}

type Payload struct {
	Header *Header 
	Data []byte 
	// ChamHash, valueOfHash equals Hash(Payload(header,data,nil))
        Chamhash []byte
}

type ProposalResponsePayload struct {
	ProposalHash []byte 
	Extension []byte
	// ChamHashStruct should be constructed in endorse statement and validated in
	// committer statement. The ChamHashStruct is same as Chamhash definition.
        ChamHashStruct []byte
}
```
### Hash Computation
We modify the hash computation of blockdata (`BlockDataHash` function in .../fabric/protoutil/blockutils.go):  

```go
func BlockDataHash(b *cb.BlockData) []byte {
	s2 := sha256.New()
	for _,env := range b.Data {
		tx := cb.Envelope{}
		_ = proto.Unmarshal(env, &tx)
		payload := cb.Payload{}
		_ = proto.Unmarshal(tx.Payload,&payload)
		PayloadHash := ChamHash.HashValueFromChamHashBytes(payload.Chamhash)
		s2.Write(PayloadHash)
		s2.Write(payload.Header.ChannelHeader)
		s2.Write(payload.Header.SignatureHeader)
		s2.Write(tx.Signature)
	}
	sum := sha256.Sum256(nil)
	return sum[:]
}
```

the hash computation of `Payload` (`FillPayload` function in .../fabric/ChamHash/EnvPayloadChamHash.go):
```go
func FillPayload(PayloadBytes []byte)([]byte, []byte){
	payload := common.Payload{}
	proto.Unmarshal(PayloadBytes,&payload)

	if payload.Chamhash != nil{
		print("exist chamstruct in PaylaodBytes\n")
		return nil,nil
	}
	chash := BytesChamHashFromBytes(PayloadBytes)
	payload.Chamhash = chash
	valuehash := HashValueFromChamHashBytes(chash)
	filledPayloadBytes,_ := proto.Marshal(&payload)
	return filledPayloadBytes, valuehash
}
```

and the hash computation of `ProposalResponsePayload` (`FillPrpStructureWithChamHash` function in .../fabric/ChamHash/PrpChamHash.go):
```go
func FillPrpStructureWithChamHash(prpbytes []byte) ([]byte, []byte){
	Prp := peer.ProposalResponsePayload{}
	err := proto.Unmarshal(prpbytes,&Prp)
	if err != nil{
		print("bad Prp bytes")
	}
	// check chamhash filled or not
	if Prp.ChamHashStruct != nil{
		return prpbytes,HashValueFromChamHashBytes(Prp.ChamHashStruct)
	}
	chambytes := BytesChamHashFromBytes(prpbytes)
	Prp.ChamHashStruct = chambytes
	realprpBytes,err := proto.Marshal(&Prp)
	if err!=nil{
		print("bad convert")
	}
	return realprpBytes,HashValueFromChamHashBytes(chambytes)
}
```

## Multi-Authority Policy-based Chameleon Hash
We use python with charm framework to implement the multi-authority policy-based chameleon hash function (MAPCH). 

The instantiation of MAPCH includes the following primitives: a multi-authority ciphertext-policy attribute-based encryption (Efficient statically-secure large-universe multi-authority attribute-based encryption, FC 2015), a chameleon hash with ephemeral trapdoor (CHET: Chameleon-hashes with ephemeral trapdoors, PKC 2017), and a symmetric encryption scheme (e.g., AES). For more details, please refer to <https://github.com/leuitong/MAPCH>.

Because the PBC Go Wrapper (<https://github.com/Nik-U/pbc>) cannot be installed, we still use python version MAPCH. We create a new folder named `ChamServer` and write a dockerfile to deploy MAPCH into docker. Note that the library of gmp, pbc, openssl and charm are not in `ChamServer`, it can be downloaded from official websites.

## Todo
world state modification & redact process
